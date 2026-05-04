---
name: relationship-loader
description: Reads all pipeline artifacts and generates two outputs per relationship type: a CSV file (for neo4j-admin bulk import) and MERGE statements appended to import.cypher (for Neo4j Browser). Relationship types: CALLS, COPIES, EXECUTES, READS, WRITES, DEFINES, CONTAINS, ENFORCED_IN, TESTS, BELONGS_TO, REDEFINES, HAS_CONDITION, RELATED_TO. Also generates cypher_library.md with 25 analytical queries and the README.md import guide. No database connection. Called after node-loader.
---

# Skill — relationship loader

## Purpose

For each relationship type, extract the relevant pairs from the pipeline
artifacts, write them as CSV rows, and append the corresponding Cypher
MERGE statements to `import.cypher`. Then generate the Cypher query
library and README import instructions.

All node MERGE statements from node-loader must already be in
`import.cypher` before this skill appends relationship statements.

---

## CSV format for relationships

All relationship CSV files follow the neo4j-admin import convention:

```
:START_ID,{properties},:END_ID,:TYPE
```

- `:START_ID` = unique ID of the source node
- `:END_ID` = unique ID of the target node
- `:TYPE` = relationship type label (e.g. CALLS)
- Properties = any additional columns

---

## Phase 1 — CALLS relationships

**Source:** `inventory_artifact.call_graph.edges`
Include edges where type = `STATIC_CALL`, `CICS_LINK`, `CICS_XCTL`
Dynamic calls become `CALLS_DYNAMIC` (see Phase 2)

### CSV file: `rels/calls.csv`

```
:START_ID,edge_type,resolved:boolean,source_line:int,cics_type,:END_ID,:TYPE
CBACT01C,STATIC_CALL,true,420,,CBACT04C,CALLS
COSGN00C,CICS_XCTL,true,312,XCTL,COADM01C,CALLS
```

For unresolved calls — still include, create stub target in Cypher:

```
CBACT01C,STATIC_CALL,false,820,,EXTPROG1,CALLS
```

### Cypher block:

```cypher
// SECTION 12 — Relationships: CALLS
// Note: unresolved targets are created as stub Program nodes

UNWIND [
  {from:'CBACT01C', to:'CBACT04C', edge_type:'STATIC_CALL', resolved:true, source_line:420, cics_type:null},
  {from:'COSGN00C', to:'COADM01C', edge_type:'CICS_XCTL',   resolved:true, source_line:312, cics_type:'XCTL'},
  {from:'CBACT01C', to:'EXTPROG1', edge_type:'STATIC_CALL', resolved:false, source_line:820, cics_type:null}
] AS row
MERGE (stub:Program {program_id: row.to}) ON CREATE SET stub.is_stub = true
WITH row, stub
MATCH (a:Program {program_id: row.from})
MERGE (a)-[r:CALLS]->(stub)
SET r.edge_type   = row.edge_type,
    r.resolved    = row.resolved,
    r.source_line = row.source_line,
    r.cics_type   = row.cics_type;
```

---

## Phase 2 — CALLS_DYNAMIC relationships

**Source:** `inventory_artifact.call_graph.edges` where type = `DYNAMIC_CALL`

Dynamic calls cannot resolve a target — create a self-loop on the source
program to record the dynamic call variable.

### CSV file: appended to `rels/calls.csv` with TYPE = `CALLS_DYNAMIC`

```
:START_ID,variable,confirmed:boolean,source_line:int,:END_ID,:TYPE
CBACT01C,WS-PROG-NAME,true,530,CBACT01C,CALLS_DYNAMIC
```

### Cypher block:

```cypher
// SECTION 13 — Relationships: CALLS_DYNAMIC (dynamic call variables)
UNWIND [
  {program_id:'CBACT01C', variable:'WS-PROG-NAME', confirmed:true, source_line:530}
] AS row
MATCH (p:Program {program_id: row.program_id})
MERGE (p)-[r:CALLS_DYNAMIC {variable: row.variable}]->(p)
SET r.confirmed   = row.confirmed,
    r.source_line = row.source_line;
```

---

## Phase 3 — COPIES relationships

**Source:** `inventory_artifact.call_graph.edges` where type = `COPY`
Also use `inventory_artifact.copybook_map` to verify

### CSV file: `rels/copies.csv`

```
:START_ID,source_line:int,resolved:boolean,:END_ID,:TYPE
CBACT01C,85,true,CVACT01Y,COPIES
CBACT01C,92,true,CVCRD01Y,COPIES
```

### Cypher block:

```cypher
// SECTION 14 — Relationships: COPIES
UNWIND [
  {program_id:'CBACT01C', copybook_id:'CVACT01Y', source_line:85, resolved:true},
  {program_id:'CBACT01C', copybook_id:'CVCRD01Y', source_line:92, resolved:true}
] AS row
MATCH (p:Program  {program_id:  row.program_id})
MATCH (c:Copybook {copybook_id: row.copybook_id})
MERGE (p)-[r:COPIES]->(c)
SET r.source_line = row.source_line,
    r.resolved    = row.resolved;
```

---

## Phase 4 — EXECUTES relationships

**Source:** `inventory_artifact.jcl_registry[].steps` where type = `cobol`

### CSV file: `rels/executes.csv`

```
:START_ID,step_name,step_order:int,:END_ID,:TYPE
ACCTJOB,ACCTPROC,2,CBACT01C,EXECUTES
```

### Cypher block:

```cypher
// SECTION 15 — Relationships: EXECUTES
UNWIND [
  {job_name:'ACCTJOB', program_id:'CBACT01C', step_name:'ACCTPROC', step_order:2}
] AS row
MATCH (j:JclJob  {job_name:   row.job_name})
MATCH (p:Program {program_id: row.program_id})
MERGE (j)-[r:EXECUTES]->(p)
SET r.step_name  = row.step_name,
    r.step_order = row.step_order;
```

---

## Phase 5 — READS and WRITES relationships

**Source:** Logic artifact pseudocode patterns (see node-loader Phase 5
for detection approach)

### CSV file: `rels/reads.csv`

```
:START_ID,access_count:int,:END_ID,:TYPE
CBACT01C,1,ACCTFILE,READS
COACT01C,1,ACCTFILE,READS
```

### CSV file: `rels/writes.csv`

```
:START_ID,access_count:int,:END_ID,:TYPE
CBACT01C,1,ACCTFILE,WRITES
```

### Cypher block:

```cypher
// SECTION 16 — Relationships: READS
UNWIND [
  {program_id:'CBACT01C', file_name:'ACCTFILE', access_count:1},
  {program_id:'COACT01C', file_name:'ACCTFILE', access_count:1}
] AS row
MATCH (p:Program {program_id: row.program_id})
MATCH (f:File    {file_name:  row.file_name})
MERGE (p)-[r:READS]->(f)
SET r.access_count = row.access_count;

// SECTION 17 — Relationships: WRITES
UNWIND [
  {program_id:'CBACT01C', file_name:'ACCTFILE', access_count:1}
] AS row
MATCH (p:Program {program_id: row.program_id})
MATCH (f:File    {file_name:  row.file_name})
MERGE (p)-[r:WRITES]->(f)
SET r.access_count = row.access_count;
```

---

## Phase 6 — DEFINES relationships

**Source:** `data_artifact.record_catalogue[]`

Copybook → Record (source_type = `copybook`)
Program → Record (source_type = `program_ws`)

### CSV file: `rels/defines.csv`

```
:START_ID,source_type,section,:END_ID,:TYPE
CVACT01Y,copybook,,REC-CVACT01Y-ACCOUNT-RECORD,DEFINES
CBACT01C,program_ws,WORKING-STORAGE,REC-CBACT01C-WS-CTRL,DEFINES
```

### Cypher block:

```cypher
// SECTION 18 — Relationships: DEFINES (Copybook → Record)
UNWIND [
  {copybook_id:'CVACT01Y', record_id:'REC-CVACT01Y-ACCOUNT-RECORD'}
] AS row
MATCH (c:Copybook {copybook_id: row.copybook_id})
MATCH (r:Record   {record_id:   row.record_id})
MERGE (c)-[:DEFINES]->(r);

// SECTION 19 — Relationships: DEFINES (Program → Record)
UNWIND [
  {program_id:'CBACT01C', record_id:'REC-CBACT01C-WS-CTRL', section:'WORKING-STORAGE'}
] AS row
MATCH (p:Program {program_id: row.program_id})
MATCH (r:Record  {record_id:  row.record_id})
MERGE (p)-[rel:DEFINES]->(r)
SET rel.section = row.section;
```

---

## Phase 7 — CONTAINS relationships

### 7a — Program → Paragraph

### CSV file: `rels/contains_prog_para.csv`

```
:START_ID,section,:END_ID,:TYPE
CBACT01C,0000-MAIN,CBACT01C::0000-MAIN,CONTAINS
CBACT01C,0000-MAIN,CBACT01C::0100-OPEN-FILES,CONTAINS
```

### Cypher block:

```cypher
// SECTION 20 — Relationships: CONTAINS (Program → Paragraph)
UNWIND [
  {program_id:'CBACT01C', para_id:'CBACT01C::0000-MAIN', section:'0000-MAIN'},
  {program_id:'CBACT01C', para_id:'CBACT01C::0100-OPEN-FILES', section:'0000-MAIN'}
] AS row
MATCH (p:Program   {program_id: row.program_id})
MATCH (q:Paragraph {para_id:    row.para_id})
MERGE (p)-[r:CONTAINS]->(q)
SET r.section = row.section;
```

### 7b — Record → Field

### CSV file: `rels/contains_rec_field.csv`

```
:START_ID,level:int,:END_ID,:TYPE
REC-CVACT01Y-ACCOUNT-RECORD,5,FLD-REC-CVACT01Y-ACCOUNT-RECORD-ACCT-ID,CONTAINS
```

### Cypher block:

```cypher
// SECTION 21 — Relationships: CONTAINS (Record → Field)
UNWIND [
  {record_id:'REC-CVACT01Y-ACCOUNT-RECORD', field_id:'FLD-REC-CVACT01Y-ACCOUNT-RECORD-ACCT-ID', level:5}
] AS row
MATCH (r:Record {record_id: row.record_id})
MATCH (f:Field  {field_id:  row.field_id})
MERGE (r)-[rel:CONTAINS]->(f)
SET rel.level = row.level;
```

---

## Phase 8 — ENFORCED_IN relationships

**Source:** `rules_artifact.business_rules[].sources[]`

`para_id` = `{program_id}::{paragraph_name}`

### CSV file: `rels/enforced_in.csv`

```
:START_ID,source_line:int,is_primary:boolean,condition_id,:END_ID,:TYPE
BR-001,680,true,COND-CBACT01C-0300-001,CBACT01C::0300-VALIDATE-AMOUNT,ENFORCED_IN
BR-001,412,false,COND-COACT02C-0250-001,COACT02C::0250-CHECK-LIMITS,ENFORCED_IN
```

### Cypher block:

```cypher
// SECTION 22 — Relationships: ENFORCED_IN
UNWIND [
  {rule_id:'BR-001', para_id:'CBACT01C::0300-VALIDATE-AMOUNT', source_line:680, is_primary:true, condition_id:'COND-CBACT01C-0300-001'},
  {rule_id:'BR-001', para_id:'COACT02C::0250-CHECK-LIMITS',    source_line:412, is_primary:false, condition_id:'COND-COACT02C-0250-001'}
] AS row
MATCH (r:BusinessRule {rule_id: row.rule_id})
MATCH (p:Paragraph    {para_id: row.para_id})
MERGE (r)-[rel:ENFORCED_IN]->(p)
SET rel.source_line  = row.source_line,
    rel.is_primary   = row.is_primary,
    rel.condition_id = row.condition_id;
```

---

## Phase 9 — TESTS relationships

**Source:** `rules_artifact.business_rules[].condition`

### CSV file: `rels/tests.csv`

```
:START_ID,role,is_subject:boolean,is_threshold:boolean,ambiguous:boolean,:END_ID,:TYPE
BR-001,SUBJECT,true,false,false,FLD-REC-CBACT01C-WS-TRANS-AMOUNT,TESTS
BR-001,THRESHOLD,false,true,false,FLD-REC-CVACT01Y-ACCOUNT-RECORD-CREDIT-LIMIT,TESTS
```

Resolve field names to field_ids from `data_artifact.field_catalogue`.
If multiple fields match the same name, set `ambiguous: true` and create
a TESTS relationship for each.

### Cypher block:

```cypher
// SECTION 23 — Relationships: TESTS
UNWIND [
  {rule_id:'BR-001', field_id:'FLD-REC-CBACT01C-WS-TRANS-AMOUNT', role:'SUBJECT', is_subject:true, is_threshold:false, ambiguous:false},
  {rule_id:'BR-001', field_id:'FLD-REC-CVACT01Y-ACCOUNT-RECORD-CREDIT-LIMIT', role:'THRESHOLD', is_subject:false, is_threshold:true, ambiguous:false}
] AS row
MATCH (r:BusinessRule {rule_id:  row.rule_id})
MATCH (f:Field        {field_id: row.field_id})
MERGE (r)-[rel:TESTS]->(f)
SET rel.role         = row.role,
    rel.is_subject   = row.is_subject,
    rel.is_threshold = row.is_threshold,
    rel.ambiguous    = row.ambiguous;
```

---

## Phase 10 — BELONGS_TO, REDEFINES, HAS_CONDITION, RELATED_TO

### CSV file: `rels/belongs_to.csv`

```
:START_ID,:END_ID,:TYPE
BR-001,RS-001,BELONGS_TO
BR-002,RS-001,BELONGS_TO
```

```cypher
// SECTION 24 — Relationships: BELONGS_TO
UNWIND [
  {rule_id:'BR-001', rule_set_id:'RS-001'}
] AS row
MATCH (r:BusinessRule {rule_id:     row.rule_id})
MATCH (rs:RuleSet     {rule_set_id: row.rule_set_id})
MERGE (r)-[:BELONGS_TO]->(rs);
```

### CSV file: `rels/redefines.csv`

```
:START_ID,bytes:int,length_mismatch:boolean,:END_ID,:TYPE
FLD-REC-CVACT01Y-ACCOUNT-RECORD-ACCT-TYPE-REDEF,3,false,FLD-REC-CVACT01Y-ACCOUNT-RECORD-ACCT-TYPE-CODE,REDEFINES
```

```cypher
// SECTION 25 — Relationships: REDEFINES
UNWIND [
  {alias_id:'FLD-REC-CVACT01Y-ACCOUNT-RECORD-ACCT-TYPE-REDEF', anchor_id:'FLD-REC-CVACT01Y-ACCOUNT-RECORD-ACCT-TYPE-CODE', bytes:3, length_mismatch:false}
] AS row
MATCH (alias:Field  {field_id: row.alias_id})
MATCH (anchor:Field {field_id: row.anchor_id})
MERGE (alias)-[r:REDEFINES]->(anchor)
SET r.bytes           = row.bytes,
    r.length_mismatch = row.length_mismatch;
```

### CSV file: `rels/has_condition.csv`

```
:START_ID,:END_ID,:TYPE
FLD-REC-CVACT01Y-ACCOUNT-RECORD-ACCT-BALANCE,COND::ACCT-OVERDRAWN::ACCT-BALANCE,HAS_CONDITION
```

```cypher
// SECTION 26 — Relationships: HAS_CONDITION
UNWIND [
  {field_id:'FLD-REC-CVACT01Y-ACCOUNT-RECORD-ACCT-BALANCE', condition_id:'COND::ACCT-OVERDRAWN::ACCT-BALANCE'}
] AS row
MATCH (f:Field     {field_id:     row.field_id})
MATCH (c:Condition {condition_id: row.condition_id})
MERGE (f)-[:HAS_CONDITION]->(c);
```

### CSV file: `rels/related_to.csv`

```
:START_ID,relationship_type,:END_ID,:TYPE
BR-001,SAME_FIELD,BR-002,RELATED_TO
```

```cypher
// SECTION 27 — Relationships: RELATED_TO
UNWIND [
  {rule_id_a:'BR-001', rule_id_b:'BR-002', relationship_type:'SAME_FIELD'}
] AS row
MATCH (a:BusinessRule {rule_id: row.rule_id_a})
MATCH (b:BusinessRule {rule_id: row.rule_id_b})
MERGE (a)-[r:RELATED_TO]->(b)
SET r.relationship_type = row.relationship_type;
```

---

## Phase 11 — cypher_library.md

After all relationship Cypher blocks are written, generate
`neo4j/cypher_library.md` with the following 25 queries:

```markdown
# Cypher query library — {SYSTEM_NAME}

Paste any query into Neo4j Browser and press Ctrl+Enter to run.

---

## Program topology

### Q01 — Most connected programs (hubs)
MATCH (caller:Program)-[:CALLS]->(hub:Program)
WITH hub, count(DISTINCT caller) AS caller_count
ORDER BY caller_count DESC LIMIT 10
RETURN hub.program_id, hub.subtype, caller_count, hub.rule_count;

### Q02 — Full call chain from a JCL job
MATCH path = (j:JclJob {job_name: 'ACCTJOB'})
             -[:EXECUTES]->(:Program)-[:CALLS*0..10]->(:Program)
RETURN [n IN nodes(path) | coalesce(n.job_name, n.program_id)] AS chain,
       length(path) AS depth ORDER BY depth;

### Q03 — Programs with no callers (entry points)
MATCH (p:Program)
WHERE NOT (:Program)-[:CALLS]->(p)
  AND NOT (:JclJob)-[:EXECUTES]->(p)
RETURN p.program_id, p.subtype, p.rule_count;

### Q04 — Programs with unstructured flow
MATCH (p:Program)
WHERE p.has_unstructured_flow = true OR p.has_alter = true
RETURN p.program_id, p.has_alter, p.has_unstructured_flow, p.rule_count
ORDER BY p.has_alter DESC;

### Q05 — Circular call chains
MATCH path = (a:Program)-[:CALLS*2..]->(a)
RETURN [n IN nodes(path) | n.program_id] AS cycle, length(path) AS depth;

---

## Copybook and data impact

### Q06 — Most widely shared copybooks
MATCH (c:Copybook)<-[:COPIES]-(p:Program)
WITH c, count(p) AS usage, collect(p.program_id) AS programs
ORDER BY usage DESC LIMIT 15
RETURN c.copybook_id, c.total_fields, usage, programs;

### Q07 — Impact if copybook changes (replace CVACT01Y)
MATCH (c:Copybook {copybook_id: 'CVACT01Y'})<-[:COPIES]-(p:Program)
OPTIONAL MATCH (p)-[:CONTAINS]->(para:Paragraph)<-[:ENFORCED_IN]-(r:BusinessRule)
RETURN p.program_id,
       count(DISTINCT para) AS paragraphs_affected,
       count(DISTINCT r) AS rules_affected
ORDER BY rules_affected DESC;

### Q08 — High-shared fields (used by 5+ programs)
MATCH (f:Field {shared_level: 'high_shared'})
MATCH (r:Record)-[:CONTAINS]->(f)
MATCH (p:Program)-[:DEFINES]->(r)
WITH f, count(DISTINCT p) AS program_count WHERE program_count >= 5
RETURN f.field_name, f.normalized_type, f.length, program_count
ORDER BY program_count DESC;

### Q09 — Where is field ACCT-BALANCE defined?
MATCH (f:Field {field_name: 'ACCT-BALANCE'})
MATCH (r:Record)-[:CONTAINS]->(f)
RETURN f.field_id, f.normalized_type, f.length, r.record_id;

### Q10 — REDEFINES groups (alternative layouts)
MATCH (alias:Field)-[rel:REDEFINES]->(anchor:Field)
MATCH (r:Record)-[:CONTAINS]->(anchor)
RETURN r.record_id,
       anchor.field_name AS storage_anchor,
       collect(alias.field_name) AS aliases,
       anchor.length AS bytes;

---

## Business rules

### Q11 — All rules for a program (replace CBACT01C)
MATCH (p:Program {program_id: 'CBACT01C'})
      -[:CONTAINS]->(para:Paragraph)<-[:ENFORCED_IN]-(r:BusinessRule)
RETURN r.rule_id, r.name, r.category, r.confidence,
       para.name AS paragraph, r.requires_sme_review
ORDER BY r.confidence DESC;

### Q12 — Rules that test a specific field (replace ACCT-BALANCE)
MATCH (r:BusinessRule)-[rel:TESTS]->(f:Field {field_name: 'ACCT-BALANCE'})
RETURN r.rule_id, r.name, r.category, r.confidence, rel.role;

### Q13 — Rules needing SME review by program
MATCH (r:BusinessRule {requires_sme_review: true})
      -[:ENFORCED_IN]->(para:Paragraph)<-[:CONTAINS]-(p:Program)
WITH p, collect({rule: r.rule_id, name: r.name}) AS rules
RETURN p.program_id, size(rules) AS review_count, rules
ORDER BY review_count DESC;

### Q14 — Duplicated rules across programs
MATCH (r:BusinessRule {is_duplicated: true})
      -[:ENFORCED_IN]->(para:Paragraph)<-[:CONTAINS]-(p:Program)
WITH r, collect(DISTINCT p.program_id) AS programs
RETURN r.rule_id, r.name, r.category, size(programs) AS count, programs
ORDER BY count DESC;

### Q15 — Rule set confidence summary
MATCH (rs:RuleSet)-[:CONTAINS]->(r:BusinessRule)
WITH rs,
     count(r) AS total,
     count(CASE WHEN r.confidence IN ['confirmed','high'] THEN 1 END) AS high_conf,
     count(CASE WHEN r.requires_sme_review THEN 1 END) AS needs_review
RETURN rs.name, total, high_conf, needs_review ORDER BY total DESC;

---

## File and I/O analysis

### Q16 — File access map
MATCH (p:Program)-[rel:READS|WRITES]->(f:File)
RETURN f.file_name, f.file_type,
       collect({program: p.program_id, access: type(rel)}) AS accessors;

### Q17 — Files written by only one program (safe to refactor)
MATCH (p:Program)-[:WRITES]->(f:File)
WITH f, count(p) AS writer_count WHERE writer_count = 1
MATCH (w:Program)-[:WRITES]->(f)
RETURN f.file_name, w.program_id AS sole_writer;

### Q18 — Producer-consumer file pairs
MATCH (producer:Program)-[:WRITES]->(f:File)<-[:READS]-(consumer:Program)
WHERE producer <> consumer
RETURN f.file_name, producer.program_id AS producer, consumer.program_id AS consumer;

---

## Complexity and risk

### Q19 — Highest complexity paragraphs
MATCH (p:Program)-[:CONTAINS]->(para:Paragraph)
WHERE para.complexity_score IS NOT NULL
RETURN p.program_id, para.name, para.complexity_score,
       para.has_goto, para.has_call
ORDER BY para.complexity_score DESC LIMIT 20;

### Q20 — Dead code candidates
MATCH (para:Paragraph {is_dead_code: true})<-[:CONTAINS]-(p:Program)
OPTIONAL MATCH (para)<-[:ENFORCED_IN]-(r:BusinessRule)
RETURN p.program_id, para.name, para.start_line,
       count(r) AS rules_in_dead_code;

### Q21 — Programs at highest modernisation risk
MATCH (p:Program)
OPTIONAL MATCH (:Program)-[:CALLS]->(p)
WITH p, count(*) AS caller_count
RETURN p.program_id,
       p.has_unstructured_flow AS goto_risk,
       p.has_alter AS alter_risk,
       p.rule_count AS business_rules,
       caller_count AS called_by,
       (CASE WHEN p.has_alter THEN 3 ELSE 0 END +
        CASE WHEN p.has_unstructured_flow THEN 2 ELSE 0 END +
        CASE WHEN caller_count > 5 THEN 2 ELSE 0 END +
        CASE WHEN p.rule_count > 10 THEN 1 ELSE 0 END) AS risk_score
ORDER BY risk_score DESC LIMIT 15;

### Q22 — 88-level coded values for a field (replace ACCT-STATUS)
MATCH (f:Field {field_name: 'ACCT-STATUS'})-[:HAS_CONDITION]->(c:Condition)
RETURN f.field_name, c.condition_name, c.values_raw;

---

## Cross-cutting

### Q23 — End-to-end: job → rules enforced
MATCH (j:JclJob {job_name: 'ACCTJOB'})
      -[:EXECUTES]->(:Program)-[:CALLS*0..5]->(p:Program)
      -[:CONTAINS]->(para:Paragraph)<-[:ENFORCED_IN]-(r:BusinessRule)
RETURN DISTINCT j.job_name, p.program_id, para.name AS paragraph,
       r.rule_id, r.name, r.category ORDER BY p.program_id;

### Q24 — Impact of changing a field type
MATCH (f:Field {field_name: 'ACCT-ID'})
MATCH (r:Record)-[:CONTAINS]->(f)
MATCH (p:Program)-[:DEFINES]->(r)
OPTIONAL MATCH (rule:BusinessRule)-[:TESTS]->(f)
RETURN f.field_name, f.normalized_type,
       collect(DISTINCT p.program_id) AS programs_to_change,
       collect(DISTINCT rule.rule_id) AS rules_to_review;

### Q25 — Test plan skeleton for a rule set
MATCH (rs:RuleSet {name: 'Credit limit rules'})
      -[:CONTAINS]->(r:BusinessRule)-[:ENFORCED_IN]->(para:Paragraph)
      <-[:CONTAINS]-(p:Program)
RETURN r.rule_id, r.name, r.condition_text, r.confidence,
       collect(DISTINCT {program: p.program_id, paragraph: para.name}) AS test_locations
ORDER BY r.category;
```

---

## Error handling

| Condition | Action |
|---|---|
| Source node ID not found in artifact | Skip relationship, log `warning` with both IDs |
| Field name resolves to multiple field_ids | Create TESTS for all, set `ambiguous:true` in CSV |
| Relationship CSV row would have empty START_ID or END_ID | Skip row, log `warning` |
| Output file not writable | Abort with clear error message |
| UNWIND block would exceed 50 items | Split into additional numbered blocks |
