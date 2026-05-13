---
name: file-walker
description: Scans a SEARCH_ROOT directory tree and builds a lookup_registry — a map of every available COBOL program, copybook, JCL job, and BMS map that can be used later for dependency resolution. Does NOT decide which files belong to the module; that is the dependency-graph skill's job. The lookup_registry is keyed by uppercased program/copybook name so references can be resolved quickly. Used exclusively by the Inventory Agent (1_inventory).
---

## Purpose

Scan the filesystem under `SEARCH_ROOT` and create a **lookup registry** of every available file. This registry acts as a resolution table: when the dependency-graph skill encounters a CALL or COPY target name, it looks the name up here to find the physical file path.

**This skill does NOT determine which files are part of the module under analysis.** It simply makes all candidate files discoverable. The
dependency-graph skill decides what is included in the final output based on reachability from the entry point.

Do NOT follow cross-references or resolve COPY/CALL statements — that is the dependency-graph skill's job.

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

If an extension does not match any row above, skip the file silently (it is
not needed for dependency resolution).

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
1. Use Glob to find all files under SEARCH_ROOT matching target extensions:
   pattern: **/*.{cbl,cob,cpy,jcl,bms,ctl,lst,sql,dclgen}
   Apply EXCLUDE_DIRS filter — skip any path segment matching an excluded name.

2. For each file found:
   a. Record absolute path and relative path from SEARCH_ROOT
   b. Determine type and subtype from extension (see table above)
   c. If type == program:
        - Extract PROGRAM-ID using grep pattern above
        - Use PROGRAM-ID (uppercased) as the lookup key
        - Also index by filename-without-extension (uppercased) as a secondary key
        - Record size_bytes
   d. If type == copybook:
        - Use filename-without-extension (uppercased) as the lookup key
   e. If type == jcl:
        - Extract JOB name
        - Extract all EXEC PGM step bindings
   f. If type == listing: record path only — do not read content

3. Build the lookup_registry as a map keyed by uppercased name:
   - programs_lookup   : { "PROGNAME": { ...entry... }, ... }
   - copybooks_lookup  : { "COPYBOOKNAME": { ...entry... }, ... }
   - jcl_lookup        : { "JOBNAME": { ...entry... }, ... }
   - bms_lookup        : { "MAPNAME": { ...entry... }, ... }

4. Return lookup_registry to the Inventory Agent.
```

---

## Output structures

### lookup_registry (returned to agent)

```json
{
  "programs_lookup": {
    "CBACT01C": {
      "id": "CBACT01C",
      "program_id": "CBACT01C",
      "path": "app/cbl/CBACT01C.CBL",
      "relative_path": "app/cbl/CBACT01C.CBL",
      "type": "program",
      "subtype": "batch",
      "size_bytes": 14200
    }
  },
  "copybooks_lookup": {
    "CVACT01Y": {
      "id": "CVACT01Y",
      "path": "app/copy/CVACT01Y.CPY",
      "relative_path": "app/copy/CVACT01Y.CPY",
      "type": "copybook"
    }
  },
  "jcl_lookup": {
    "ACCTFILE": {
      "id": "ACCTFILE",
      "path": "app/jcl/ACCTFILE.JCL",
      "relative_path": "app/jcl/ACCTFILE.JCL",
      "job_name": "ACCTFILE",
      "steps": [
        { "step_name": "SORT001",  "program": "SORT",     "type": "utility" },
        { "step_name": "ACCTPROC", "program": "CBACT01C", "type": "cobol"   }
      ]
    }
  },
  "bms_lookup": {
    "INQSET": {
      "id": "INQSET",
      "path": "maps/INQSET.BMS",
      "relative_path": "maps/INQSET.BMS",
      "type": "bms_map"
    }
  }
}
```

Step `type` rules for JCL: if program name is `SORT`, `IDCAMS`, `IEBGENER`,
`IEFBR14`, `DFSRRC00` → `utility`; else → `cobol`.

---

## Error handling

| Condition | Action |
|---|---|
| File cannot be read | Log `error` issue, continue to next file |
| PROGRAM-ID not found in `.cbl` | Use filename as ID, log `warning` |
| Duplicate PROGRAM-ID across two files | Log `error` with both paths — keep both, second entry stored under filename-key |
| JOB name not found in `.jcl` | Use filename as ID, log `warning` |
