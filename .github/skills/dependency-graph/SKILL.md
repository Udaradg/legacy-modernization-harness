---
name: dependency-graph
description: >
  Reads the file_registry produced by the file-walker skill. Opens each
  program file and scans for all outgoing cross-references: COPY statements,
  static and dynamic CALLs, CICS LINK/XCTL commands, and SQL INCLUDEs.
  Builds a directed dependency graph (nodes + typed edges), populates
  copybook_map and used_by fields, and collects all issues. Produces the
  final inventory_artifact.json schema sections: call_graph, copybook_map,
  stats, and issues. Used exclusively by the Inventory Agent (1_inventory).
---

## Purpose

For every program in the file_registry, detect all outgoing references to
other programs and copybooks. Build a directed graph. Populate the
`used_by` field in the copybook_registry. Collect unresolved references and
dynamic calls as issues.

Do NOT read or interpret COBOL logic, paragraph names, or data definitions.
Only scan for cross-reference statements.

---

## Reference types to detect

| Statement pattern | Edge type | Notes |
|---|---|---|
| `COPY XXXX` | `COPY` | Standard copybook inclusion |
| `COPY XXXX IN MYLIB` | `COPY` | Strip library qualifier — use XXXX only |
| `COPY XXXX OF MYLIB` | `COPY` | Strip library qualifier — use XXXX only |
| `CALL 'PROGNAME'` | `STATIC_CALL` | Literal string — target is known |
| `CALL "PROGNAME"` | `STATIC_CALL` | Same, double-quoted variant |
| `CALL WS-VARIABLE` | `DYNAMIC_CALL` | Variable — target cannot be resolved statically |
| `EXEC CICS LINK PROGRAM(...)` | `CICS_LINK` | CICS synchronous subprogram call |
| `EXEC CICS XCTL PROGRAM(...)` | `CICS_XCTL` | CICS transfer of control (no return) |
| `EXEC CICS RETURN TRANSID(...)` | `CICS_RETURN` | Records transaction ID — no target program |
| `EXEC SQL INCLUDE XXXX` | `SQL_INCLUDE` | DB2 DCLGEN or SQL copybook |

---

## Grep patterns

Apply these patterns per program file. Always respect the COBOL column rules
from the file-walker skill: skip lines where column 7 (index 6) is `*` or `/`.

```bash
# COPY — basic and with library qualifier
grep -niE "^\s{0,6}[^*/].{0,1}\s+COPY\s+([A-Z0-9\-]+)" "$FILE"

# COPY IN / OF (library-qualified) — capture group 1 = copybook name
grep -niE "COPY\s+([A-Z0-9\-]+)\s+(IN|OF)\s+[A-Z0-9\-]+" "$FILE"

# CALL with single-quoted literal
grep -niE "CALL\s+'([A-Z0-9\-]+)'" "$FILE"

# CALL with double-quoted literal
grep -niE "CALL\s+\"([A-Z0-9\-]+)\"" "$FILE"

# CALL with variable (dynamic) — capture the variable name
grep -niE "CALL\s+([A-Z][A-Z0-9\-]{2,})\b" "$FILE"

# EXEC CICS LINK — program name in quotes or bare
grep -niE "EXEC\s+CICS\s+LINK\s+PROGRAM\s*\(\s*['\"]?([A-Z0-9\-]+)" "$FILE"

# EXEC CICS XCTL
grep -niE "EXEC\s+CICS\s+XCTL\s+PROGRAM\s*\(\s*['\"]?([A-Z0-9\-]+)" "$FILE"

# EXEC CICS RETURN with TRANSID
grep -niE "EXEC\s+CICS\s+RETURN\s+TRANSID\s*\(\s*['\"]?([A-Z0-9\-]+)" "$FILE"

# EXEC SQL INCLUDE
grep -niE "EXEC\s+SQL\s+INCLUDE\s+([A-Z0-9\-]+)" "$FILE"
```

The `-n` flag captures line numbers — store them as `source_line_hint` on
each edge.

---

## Dynamic CALL disambiguation

A `CALL variable-name` match may look like a dynamic call but could be a
false positive if the variable name is a keyword or very short. Apply these
filters before classifying as `DYNAMIC_CALL`:

- Variable name must be 3+ characters
- Must NOT be: `USING`, `BY`, `REFERENCE`, `CONTENT`, `VALUE`, `LENGTH`
- Must start with a letter
- Must be present in the WORKING-STORAGE SECTION of the same file
  (use a secondary grep to confirm — if not found, still record but flag
  `"confirmed": false`)

---

## Graph construction rules

```
1. Start with nodes from file_registry (programs) and copybook_registry
2. For each program, run all grep patterns above
3. For each match:
   a. Normalise target name to UPPERCASE
   b. Look up target in file_registry or copybook_registry
   c. If found: set "resolved": true
   d. If not found: set "resolved": false, log warning issue
   e. Create edge object (see schema below)
4. For COPY edges: add source program ID to copybook_registry[target].used_by
5. For DYNAMIC_CALL: set "target": null, record variable name
6. Detect circular COPY chains: if A COPYs B and B COPYs A → log error issue
7. After all files processed: compute stats block
```

---

## Output specification

This skill produces the following sections of `inventory_artifact.json`:

### Full artifact schema

```json
{
  "meta": {
    "generated_at": "2024-01-15T10:30:00Z",
    "repo_root": "/absolute/path/to/repo",
    "agent_version": "1_inventory@1.0",
    "total_files_scanned": 85
  },
  "stats": {
    "programs": 42,
    "batch_programs": 18,
    "online_programs": 24,
    "copybooks": 31,
    "jcl_jobs": 12,
    "bms_maps": 8,
    "control_cards": 2,
    "db2_includes": 4,
    "call_edges_total": 187,
    "call_edges_resolved": 179,
    "call_edges_unresolved": 8,
    "dynamic_calls": 4,
    "copy_edges": 143,
    "cics_link_edges": 22,
    "cics_xctl_edges": 6
  },
  "file_registry": [ ],
  "copybook_registry": [ ],
  "jcl_registry": [ ],
  "call_graph": {
    "nodes": [
      {
        "id": "CBACT01C",
        "type": "program",
        "subtype": "batch",
        "path": "app/cbl/CBACT01C.CBL"
      },
      {
        "id": "CVACT01Y",
        "type": "copybook",
        "path": "app/copy/CVACT01Y.CPY"
      }
    ],
    "edges": [
      {
        "from": "CBACT01C",
        "to": "CBACT04C",
        "type": "STATIC_CALL",
        "resolved": true,
        "source_line_hint": 420
      },
      {
        "from": "CBACT01C",
        "to": "CVACT01Y",
        "type": "COPY",
        "resolved": true,
        "source_line_hint": 85
      },
      {
        "from": "CBACT01C",
        "to": null,
        "type": "DYNAMIC_CALL",
        "resolved": false,
        "variable": "WS-PROG-NAME",
        "confirmed": true,
        "source_line_hint": 530
      },
      {
        "from": "COSGN00C",
        "to": "COADM01C",
        "type": "CICS_XCTL",
        "resolved": true,
        "source_line_hint": 312
      }
    ]
  },
  "copybook_map": {
    "CVACT01Y": ["CBACT01C", "COACT01C", "COACT02C"],
    "CVCRD01Y": ["CBACT01C", "COCRDSLC", "COCRDUPC"]
  },
  "issues": [
    {
      "severity": "warning",
      "type": "unresolved_reference",
      "source": "CBACT01C",
      "reference": "EXTPROG1",
      "edge_type": "STATIC_CALL",
      "message": "CALL target 'EXTPROG1' not found in file registry — may be external or in load library"
    },
    {
      "severity": "info",
      "type": "dynamic_call",
      "source": "CBACT01C",
      "variable": "WS-PROG-NAME",
      "confirmed": true,
      "message": "Dynamic CALL via variable WS-PROG-NAME — cannot resolve statically"
    },
    {
      "severity": "warning",
      "type": "missing_program_id",
      "source_file": "app/cbl/OLDPROG.CBL",
      "fallback_id": "OLDPROG",
      "message": "PROGRAM-ID not found — using filename as fallback ID"
    },
    {
      "severity": "error",
      "type": "circular_copy",
      "source": "COPYBA",
      "target": "COPYAB",
      "message": "Circular COPY chain detected: COPYBA COPY COPYAB which COPYs COPYBA"
    }
  ]
}
```

---

## Error handling

| Condition | Severity | Action |
|---|---|---|
| Target of CALL/COPY not in registry | `warning` | Add edge with `resolved: false`, log issue |
| Dynamic CALL variable not confirmed in WS | `info` | Add edge with `confirmed: false`, log issue |
| Circular COPY chain | `error` | Log issue, do not recurse — record both parties |
| CICS LINK/XCTL target not in registry | `warning` | Add edge with `resolved: false`, log issue |
| Grep produces no output for a file | `info` | File has no outgoing references — normal for some utilities |
| File in file_registry cannot be re-read | `error` | Log issue, skip edge extraction for that file |
