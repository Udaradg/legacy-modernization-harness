---
name: node-loader
description: >
  Reads all pipeline artifacts and generates two outputs per node type:
  a CSV file (for neo4j-admin bulk import) and MERGE statements written
  into import.cypher (for Neo4j Browser). Node types: Program, Copybook,
  JclJob, File, Record, Field, Paragraph, BusinessRule, RuleSet, Condition.
  No database connection. All output is plain text files. Called by the
  Graph Agent before relationship-loader.
tools: Read, Bash
---

# Skill — node loader

## Purpose

For each node type, extract records from the pipeline artifacts, write
them as CSV rows, and write the corresponding Cypher MERGE statements
into `import.cypher`. All string escaping must be applied before writing.

---

## String escaping rules

Apply these to every string value before writing to CSV or Cypher:

**For CSV fields:**
- If value contains a comma, newline, or double quote → wrap in `"..."`
- If value contains a double quote → replace with `""`
- Null/missing values → write as empty string (no quotes)
- Truncate any single field value to 5000 characters maximum

**For Cypher string literals:**
- Replace every `'` with `\'`
- Replace every `\` with `\\` (before the quote replacement)
- Replace every newline with ` ` (space)
- Truncate any single property value to 5000 characters maximum

---

## Phase 1 — initialise output files

```bash
mkdir -p "$OUTPUT_DIR/neo4j/nodes"
mkdir -p "$OUTPUT_DIR/neo4j/rels"

# Create import.cypher with header
cat > "$OUTPUT_DIR/neo4j/import.cypher" << 'EOF'
// ============================================================
// COBOL Reverse Engineering Graph — Import Script
// Run this file in Neo4j Browser (top to bottom)
// Use :auto prefix blocks for large datasets
// ============================================================

// SECTION 1 — Schema constraints and indexes
CREATE CONSTRAINT prog_id   IF NOT EXISTS FOR (n:Program)      REQUIRE n.program_id   IS UNIQUE;
CREATE CONSTRAINT copy_id   IF NOT EXISTS FOR (n:Copybook)     REQUIRE n.copybook_id  IS UNIQUE;
CREATE CONSTRAINT jcl_id    IF NOT EXISTS FOR (n:JclJob)       REQUIRE n.job_name     IS UNIQUE;
CREATE CONSTRAINT file_id   IF NOT EXISTS FOR (n:File)         REQUIRE n.file_name    IS UNIQUE;
CREATE CONSTRAINT rec_id    IF NOT EXISTS FOR (n:Record)       REQUIRE n.record_id    IS UNIQUE;
CREATE CONSTRAINT field_id  IF NOT EXISTS FOR (n:Field)        REQUIRE n.field_id     IS UNIQUE;
CREATE CONSTRAINT para_id   IF NOT EXISTS FOR (n:Paragraph)    REQUIRE n.para_id      IS UNIQUE;
CREATE CONSTRAINT rule_id   IF NOT EXISTS FOR (n:BusinessRule) REQUIRE n.rule_id      IS UNIQUE;
CREATE CONSTRAINT rset_id   IF NOT EXISTS FOR (n:RuleSet)      REQUIRE n.rule_set_id  IS UNIQUE;
CREATE CONSTRAINT cond_id   IF NOT EXISTS FOR (n:Condition)    REQUIRE n.condition_id IS UNIQUE;

CREATE INDEX prog_subtype IF NOT EXISTS FOR (n:Program)      ON (n.subtype);
CREATE INDEX rule_conf    IF NOT EXISTS FOR (n:BusinessRule) ON (n.confidence);
CREATE INDEX rule_cat     IF NOT EXISTS FOR (n:BusinessRule) ON (n.category);
CREATE INDEX para_complex IF NOT EXISTS FOR (n:Paragraph)   ON (n.complexity_score);

EOF
```

---

## Phase 2 — Program nodes

**Source:** `inventory_artifact.file_registry[]`
Cross-reference with `parser_artifact.programs[]` and `rules_artifact.rules_by_program`

### CSV file: `nodes/programs.csv`

Header row:
```
program_id,path,subtype,size_bytes,total_lines,paragraph_count,section_count,has_unstructured_flow,has_alter,dead_code_candidates,parse_status,rule_count,complexity_label,program_narrative
```

Derive `complexity_label` from average paragraph complexity score:
- 1.0–3.9 → `Low`
- 4.0–6.9 → `Medium`
- 7.0+ → `High`
- Not available → `Unknown`

`program_narrative` = first 500 characters of narrative from logic artifact, newlines replaced with space.

### Cypher block appended to `import.cypher`:

```cypher
// SECTION 2 — Nodes: Programs (42 nodes)
UNWIND [
  {program_id:'CBACT01C', path:'app/cbl/CBACT01C.CBL', subtype:'batch', size_bytes:14200, total_lines:892, paragraph_count:24, section_count:5, has_unstructured_flow:true, has_alter:false, dead_code_candidates:0, parse_status:'success', rule_count:12, complexity_label:'High', program_narrative:'CBACT01C is a batch program that processes account transactions...'},
  {program_id:'COSGN00C', path:'app/cbl/COSGN00C.CBL', subtype:'online', size_bytes:6420, total_lines:410, paragraph_count:8, section_count:2, has_unstructured_flow:false, has_alter:false, dead_code_candidates:0, parse_status:'success', rule_count:3, complexity_label:'Low', program_narrative:'COSGN00C handles the CICS sign-on screen...'}
] AS row
MERGE (p:Program {program_id: row.program_id})
SET p.path                  = row.path,
    p.subtype               = row.subtype,
    p.size_bytes            = row.size_bytes,
    p.total_lines           = row.total_lines,
    p.paragraph_count       = row.paragraph_count,
    p.section_count         = row.section_count,
    p.has_unstructured_flow = row.has_unstructured_flow,
    p.has_alter             = row.has_alter,
    p.dead_code_candidates  = row.dead_code_candidates,
    p.parse_status          = row.parse_status,
    p.rule_count            = row.rule_count,
    p.complexity_label      = row.complexity_label,
    p.program_narrative     = row.program_narrative;
```

Split into batches of 50 programs per UNWIND block if total > 50.

---

## Phase 3 — Copybook nodes

**Source:** `inventory_artifact.copybook_registry[]`
Cross-reference with `data_artifact.record_catalogue[]`

### CSV file: `nodes/copybooks.csv`

Header:
```
copybook_id,path,total_fields,total_bytes,has_redefines,has_occurs,used_by_count
```

### Cypher block:

```cypher
// SECTION 3 — Nodes: Copybooks (31 nodes)
UNWIND [
  {copybook_id:'CVACT01Y', path:'app/copy/CVACT01Y.CPY', total_fields:24, total_bytes:300, has_redefines:true, has_occurs:false, used_by_count:3},
  {copybook_id:'CVCRD01Y', path:'app/copy/CVCRD01Y.CPY', total_fields:18, total_bytes:200, has_redefines:false, has_occurs:false, used_by_count:5}
] AS row
MERGE (c:Copybook {copybook_id: row.copybook_id})
SET c.path          = row.path,
    c.total_fields  = row.total_fields,
    c.total_bytes   = row.total_bytes,
    c.has_redefines = row.has_redefines,
    c.has_occurs    = row.has_occurs,
    c.used_by_count = row.used_by_count;
```

---

## Phase 4 — JclJob nodes

**Source:** `inventory_artifact.jcl_registry[]`

### CSV file: `nodes/jcl_jobs.csv`

Header:
```
job_name,path,step_count,cobol_programs,datasets
```

`cobol_programs` = pipe-separated list of program IDs from steps where type = `cobol`
`datasets` = pipe-separated list from `datasets_referenced`

### Cypher block:

```cypher
// SECTION 4 — Nodes: JCL jobs (12 nodes)
UNWIND [
  {job_name:'ACCTJOB', path:'app/jcl/ACCTFILE.JCL', step_count:3, cobol_programs:'CBACT01C|CBUTL01C', datasets:'ACCT.MASTER|ACCT.TRANS'}
] AS row
MERGE (j:JclJob {job_name: row.job_name})
SET j.path           = row.path,
    j.step_count     = row.step_count,
    j.cobol_programs = row.cobol_programs,
    j.datasets       = row.datasets;
```

---

## Phase 5 — File nodes

**Source:** `inventory_artifact.jcl_registry[].datasets_referenced`
and OPEN/CICS patterns from `logic_artifact` pseudocode

Collect all unique file/dataset names across the codebase.
Infer file_type:
- Contains `READ ... KEY` or `INVALID KEY` in pseudocode → `VSAM`
- Contains `WRITE ... ADVANCING` → `REPORT`
- Used in SORT step in JCL → `SORT`
- Default → `SEQUENTIAL`

### CSV file: `nodes/files.csv`

Header:
```
file_name,file_type,used_by_count,access_modes
```

`access_modes` = pipe-separated list of unique modes: `READ`, `WRITE`, `READ_WRITE`

### Cypher block:

```cypher
// SECTION 5 — Nodes: Files (18 nodes)
UNWIND [
  {file_name:'ACCTFILE', file_type:'VSAM', used_by_count:4, access_modes:'READ|WRITE'},
  {file_name:'CARDFILE', file_type:'VSAM', used_by_count:2, access_modes:'READ'}
] AS row
MERGE (f:File {file_name: row.file_name})
SET f.file_type     = row.file_type,
    f.used_by_count = row.used_by_count,
    f.access_modes  = row.access_modes;
```

---

## Phase 6 — Record nodes

**Source:** `data_artifact.record_catalogue[]`

### CSV file: `nodes/records.csv`

Header:
```
record_id,record_name,source_id,source_type,total_fields,total_bytes,has_redefines,has_occurs
```

### Cypher block:

```cypher
// SECTION 6 — Nodes: Records (187 nodes)
UNWIND [
  {record_id:'REC-CVACT01Y-ACCOUNT-RECORD', record_name:'ACCOUNT-RECORD', source_id:'CVACT01Y', source_type:'copybook', total_fields:24, total_bytes:300, has_redefines:true, has_occurs:false}
] AS row
MERGE (r:Record {record_id: row.record_id})
SET r.record_name   = row.record_name,
    r.source_id     = row.source_id,
    r.source_type   = row.source_type,
    r.total_fields  = row.total_fields,
    r.total_bytes   = row.total_bytes,
    r.has_redefines = row.has_redefines,
    r.has_occurs    = row.has_occurs;
```

---

## Phase 7 — Field nodes

**Source:** `data_artifact.field_catalogue[]`

Large dataset — always split into batches of 50 per UNWIND block in Cypher.
Write full dataset to CSV regardless.

### CSV file: `nodes/fields.csv`

Header:
```
field_id,record_id,field_name,qualified_name,level,normalized_type,length,decimal_places,signed,storage_type,pic_raw,is_filler,is_condition,occurs_max,shared_level,source_line
```

### Cypher block (batched):

```cypher
// SECTION 7 — Nodes: Fields (2341 nodes, split into batches of 50)

// Batch 1 of 47
UNWIND [
  {field_id:'FLD-REC-CVACT01Y-ACCOUNT-RECORD-ACCT-ID', record_id:'REC-CVACT01Y-ACCOUNT-RECORD', field_name:'ACCT-ID', qualified_name:'ACCOUNT-RECORD.ACCT-ID', level:5, normalized_type:'STRING', length:11, decimal_places:0, signed:false, storage_type:'DISPLAY', pic_raw:'X(11)', is_filler:false, is_condition:false, occurs_max:null, shared_level:'high_shared', source_line:7}
] AS row
MERGE (f:Field {field_id: row.field_id})
SET f.record_id       = row.record_id,
    f.field_name      = row.field_name,
    f.qualified_name  = row.qualified_name,
    f.level           = row.level,
    f.normalized_type = row.normalized_type,
    f.length          = row.length,
    f.decimal_places  = row.decimal_places,
    f.signed          = row.signed,
    f.storage_type    = row.storage_type,
    f.pic_raw         = row.pic_raw,
    f.is_filler       = row.is_filler,
    f.is_condition    = row.is_condition,
    f.occurs_max      = row.occurs_max,
    f.shared_level    = row.shared_level,
    f.source_line     = row.source_line;

// Batch 2 of 47
UNWIND [...] AS row
MERGE (f:Field {field_id: row.field_id})
SET f += row;
```

---

## Phase 8 — Paragraph nodes

**Source:** `parser_artifact.raw_structure/{program_id}.json`
Cross-reference with `logic_artifact.program_logic/{program_id}_logic.json`

`para_id` format: `{program_id}::{paragraph_name}`

### CSV file: `nodes/paragraphs.csv`

Header:
```
para_id,program_id,name,section,start_line,end_line,complexity_score,has_perform,has_goto,has_call,has_stop_run,is_entry_point,is_terminal,is_dead_code,pseudocode_preview
```

`pseudocode_preview` = first 300 characters of joined pseudocode lines

### Cypher block:

```cypher
// SECTION 8 — Nodes: Paragraphs (1847 nodes, batched)
UNWIND [
  {para_id:'CBACT01C::0000-MAIN', program_id:'CBACT01C', name:'0000-MAIN', section:'0000-MAIN', start_line:425, end_line:480, complexity_score:2, has_perform:true, has_goto:false, has_call:false, has_stop_run:false, is_entry_point:true, is_terminal:false, is_dead_code:false, pseudocode_preview:'CALL 0100-open-files\nCALL 0150-initialise-counters...'}
] AS row
MERGE (p:Paragraph {para_id: row.para_id})
SET p.program_id       = row.program_id,
    p.name             = row.name,
    p.section          = row.section,
    p.start_line       = row.start_line,
    p.end_line         = row.end_line,
    p.complexity_score = row.complexity_score,
    p.has_perform      = row.has_perform,
    p.has_goto         = row.has_goto,
    p.has_call         = row.has_call,
    p.has_stop_run     = row.has_stop_run,
    p.is_entry_point   = row.is_entry_point,
    p.is_terminal      = row.is_terminal,
    p.is_dead_code     = row.is_dead_code,
    p.pseudocode_preview = row.pseudocode_preview;
```

---

## Phase 9 — BusinessRule nodes

**Source:** `rules_artifact.business_rules[]`

### CSV file: `nodes/business_rules.csv`

Header:
```
rule_id,name,rule_set_id,category,confidence,requires_sme_review,sme_review_note,description,condition_text,pattern,is_duplicated,primary_program,primary_paragraph,primary_line
```

### Cypher block:

```cypher
// SECTION 9 — Nodes: BusinessRules (203 nodes)
UNWIND [
  {rule_id:'BR-001', name:'Enforce transaction amount does not exceed credit limit', rule_set_id:'RS-001', category:'LIMIT_CHECK', confidence:'confirmed', requires_sme_review:false, sme_review_note:'', description:'When the transaction amount exceeds the credit limit...', condition_text:'WS-TRANS-AMOUNT > WS-CREDIT-LIMIT', pattern:'FIELD_FIELD_COMPARE', is_duplicated:true, primary_program:'CBACT01C', primary_paragraph:'0300-VALIDATE-AMOUNT', primary_line:680}
] AS row
MERGE (r:BusinessRule {rule_id: row.rule_id})
SET r.name                = row.name,
    r.rule_set_id         = row.rule_set_id,
    r.category            = row.category,
    r.confidence          = row.confidence,
    r.requires_sme_review = row.requires_sme_review,
    r.sme_review_note     = row.sme_review_note,
    r.description         = row.description,
    r.condition_text      = row.condition_text,
    r.pattern             = row.pattern,
    r.is_duplicated       = row.is_duplicated,
    r.primary_program     = row.primary_program,
    r.primary_paragraph   = row.primary_paragraph,
    r.primary_line        = row.primary_line;
```

---

## Phase 10 — RuleSet nodes

**Source:** `rules_artifact.rule_sets[]`

### CSV file: `nodes/rule_sets.csv`

Header:
```
rule_set_id,name,description,rule_count
```

### Cypher block:

```cypher
// SECTION 10 — Nodes: RuleSets (12 nodes)
UNWIND [
  {rule_set_id:'RS-001', name:'Credit limit rules', description:'Rules that enforce transaction and balance limits.', rule_count:5}
] AS row
MERGE (rs:RuleSet {rule_set_id: row.rule_set_id})
SET rs.name        = row.name,
    rs.description = row.description,
    rs.rule_count  = row.rule_count;
```

---

## Phase 11 — Condition nodes (88-level)

**Source:** `data_artifact.condition_catalogue[]`

`condition_id` format: `COND::{condition_name}::{parent_field}`

### CSV file: `nodes/conditions.csv`

Header:
```
condition_id,condition_name,parent_field,parent_record,values_raw,programs_in_scope,condition_group_size
```

`values_raw` = JSON-serialised value array, escaped for CSV
`programs_in_scope` = pipe-separated program IDs

### Cypher block:

```cypher
// SECTION 11 — Nodes: Conditions (312 nodes)
UNWIND [
  {condition_id:'COND::ACCT-OVERDRAWN::ACCT-BALANCE', condition_name:'ACCT-OVERDRAWN', parent_field:'ACCT-BALANCE', parent_record:'REC-CVACT01Y-ACCOUNT-RECORD', values_raw:'[{"from":-999999999.99,"to":-0.01}]', programs_in_scope:'CBACT01C|COACT01C', condition_group_size:4}
] AS row
MERGE (c:Condition {condition_id: row.condition_id})
SET c.condition_name       = row.condition_name,
    c.parent_field         = row.parent_field,
    c.parent_record        = row.parent_record,
    c.values_raw           = row.values_raw,
    c.programs_in_scope    = row.programs_in_scope,
    c.condition_group_size = row.condition_group_size;
```

---

## Error handling

| Condition | Action |
|---|---|
| Field value is null in artifact | Write empty string in CSV, omit property from Cypher SET |
| Field value contains newline | Replace with space before writing |
| Program in parser_artifact not in inventory | Include in CSV/Cypher with available properties, log `warning` |
| Cypher UNWIND batch exceeds 50 items | Split into additional UNWIND blocks with incrementing batch comments |
| Output directory not writable | Abort with clear error message |
| Single string value exceeds 5000 chars | Truncate and append `[TRUNCATED]`, log `info` |
