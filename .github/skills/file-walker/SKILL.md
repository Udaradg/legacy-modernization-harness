---
name: file-walker
description: >
  Recursively traverses a COBOL repository directory tree. Classifies every
  file by type based on its extension, extracts PROGRAM-ID from COBOL source
  files, and produces the raw file_registry, copybook_registry, and
  jcl_registry that the Inventory Agent assembles into the final artifact.
  Used exclusively by the Inventory Agent (1_inventory).
---

## Purpose

Walk the repository filesystem. For every file found, record its path,
classify its type, and extract key identifiers (PROGRAM-ID for programs,
JOB name for JCL). Produce structured registry lists.

Do NOT follow cross-references or resolve COPY/CALL statements — that is
the dependency-graph skill's job.

---

## File type classification

| Extension | Type | Subtype logic |
|---|---|---|
| `.cbl`, `.cob` | `program` | Subtype: `batch` if filename starts with `CB`; `online` if starts with `CO` or `CA`; else `unknown` |
| `.cpy` | `copybook` | — |
| `.jcl` | `jcl` | — |
| `.bms` | `bms_map` | CICS screen map |
| `.ctl` | `control_card` | Sort/utility control |
| `.lst` | `listing` | Compiler listing — record path only, skip content scan |
| `.sql`, `.dclgen` | `db2` | DB2 DCLGEN or embedded SQL include |

If an extension does not match any row above, record it with type `unknown`
and log an `info` issue.

---

## COBOL fixed-format column rules

COBOL source files use fixed-column layout. Always apply these rules when
reading any `.cbl` or `.cob` file:

| Columns | Content | Action |
|---|---|---|
| 1–6 | Sequence numbers | Ignore |
| 7 | Indicator | If `*` or `/` → skip this line (comment). If `D` → include (debug line). |
| 8–72 | Program text | Read and scan this range only |
| 73–80 | Identification | Ignore |

In 0-indexed terms: `line[6]` is the indicator, `line[7:72]` is program text.

---

## PROGRAM-ID extraction

For every `.cbl` and `.cob` file, find the PROGRAM-ID using this approach:

```bash
# Grep for PROGRAM-ID, skipping comment lines (column 7 = *)
grep -iE "^.{6}[^*/].{0,1}\s*PROGRAM-ID\.\s+([A-Z0-9\-]+)" "$FILE" | head -1
```

Extraction rules:
- Match is case-insensitive
- Strip trailing period from the extracted name
- Strip surrounding whitespace
- If no PROGRAM-ID is found, fall back to the filename without extension
  (uppercased) and log a `warning` issue

---

## JCL JOB name extraction

For every `.jcl` file, extract the JOB card name:

```bash
# First line starting with //JOBNAME JOB
grep -iE "^//([A-Z0-9#@\$]{1,8})\s+JOB\s" "$FILE" | head -1
```

Also extract all EXEC PGM steps:

```bash
# All EXEC PGM lines — captures stepname and program name
grep -iE "^//([A-Z0-9#@\$]{1,8})\s+EXEC\s+PGM=([A-Z0-9\-]+)" "$FILE"
```

---

## Execution steps

```
1. Use Glob to find all files under REPO_ROOT matching target extensions:
   pattern: **/*.{cbl,cob,cpy,jcl,bms,ctl,lst,sql,dclgen}
   Apply EXCLUDE_DIRS filter — skip any path containing an excluded segment.

2. For each file found:
   a. Record absolute path and relative path from REPO_ROOT
   b. Determine type and subtype from extension (see table above)
   c. If type == program:
        - Extract PROGRAM-ID using grep pattern above
        - Record size_bytes
   d. If type == jcl:
        - Extract JOB name
        - Extract all EXEC PGM step bindings
   e. If type == listing: record path only — do not read content

3. Build three separate lists:
   - file_registry      (programs)
   - copybook_registry  (copybooks and db2 includes)
   - jcl_registry       (jcl jobs with their steps)

4. Return all three lists to the Inventory Agent.
```

---

## Output structures

### file_registry entry

```json
{
  "id": "CBACT01C",
  "program_id": "CBACT01C",
  "path": "app/cbl/CBACT01C.CBL",
  "relative_path": "app/cbl/CBACT01C.CBL",
  "type": "program",
  "subtype": "batch",
  "size_bytes": 14200
}
```

### copybook_registry entry

```json
{
  "id": "CVACT01Y",
  "path": "app/copy/CVACT01Y.CPY",
  "relative_path": "app/copy/CVACT01Y.CPY",
  "type": "copybook",
  "used_by": []
}
```

Note: `used_by` starts empty here — the dependency-graph skill populates it.

### jcl_registry entry

```json
{
  "id": "ACCTFILE",
  "path": "app/jcl/ACCTFILE.JCL",
  "relative_path": "app/jcl/ACCTFILE.JCL",
  "job_name": "ACCTFILE",
  "steps": [
    { "step_name": "SORT001", "program": "SORT",     "type": "utility" },
    { "step_name": "ACCTPROC","program": "CBACT01C", "type": "cobol"   }
  ]
}
```

Step `type` rules: if program name is `SORT`, `IDCAMS`, `IEBGENER`,
`IEFBR14`, `DFSRRC00` → `utility`; else → `cobol`.

---

## Error handling

| Condition | Action |
|---|---|
| File cannot be read | Log `error` issue, continue to next file |
| PROGRAM-ID not found in `.cbl` | Use filename as ID, log `warning` |
| Duplicate PROGRAM-ID across two files | Log `error` with both paths — do not deduplicate |
| JOB name not found in `.jcl` | Use filename as ID, log `warning` |
| File extension not in classification table | Record as `unknown` type, log `info` |
