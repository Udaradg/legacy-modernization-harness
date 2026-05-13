---
name: dependency-graph
description: Receives the lookup_registry produced by the file-walker skill and an ENTRY_POINT_FILE. Starting from the entry point, performs a breadth-first traversal of all COPY, static/dynamic CALL, CICS LINK/XCTL, and SQL INCLUDE references. Only files reachable (directly or transitively) from the entry point are included in the final output. Unreachable .cbl files that exist in the search root are silently excluded. Builds a directed dependency graph, populates copybook_map and used_by fields, and collects all issues. Produces the final inventory_artifact.json. Used exclusively by the Inventory Agent (1_inventory).
---

## Purpose

Starting from `ENTRY_POINT_FILE`, discover the complete dependency tree by scanning each reachable program file for outgoing cross-references and following them recursively. Build a directed graph containing **only** the programs and copybooks that are reachable from the entry point.

Files that exist in `SEARCH_ROOT` but are NOT reachable from the entry point must **not** appear in the output `file_registry`, `copybook_registry`, or `call_graph`.

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

Apply these patterns per program file. Always respect the COBOL column rules from the file-walker skill: skip lines where column 7 (index 6) is `*` or `/`.

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

## BFS traversal algorithm

```
INPUTS:
  entry_point_file  — absolute path to the root .cbl file
  lookup_registry   — { programs_lookup, copybooks_lookup, jcl_lookup, bms_lookup }

OUTPUTS (built incrementally):
  file_registry     — list of reachable program entries (initially empty)
  copybook_registry — list of reachable copybook entries (initially empty)
  jcl_registry      — list of JCL jobs that reference reachable programs
  call_graph.nodes  — de-duplicated node list (program + copybook)
  call_graph.edges  — all edges found during traversal
  copybook_map      — { copybook_name: [program_ids_that_copy_it] }
  issues[]          — collected warnings and errors

ALGORITHM:
  visited_programs  = empty set    # program IDs already processed
  visited_copybooks = empty set    # copybook IDs already processed
  queue             = [ entry_point_file ]   # BFS work queue (program files only)

  STEP 1 — Seed:
    Extract PROGRAM-ID from entry_point_file (same grep as file-walker).
    Create a file_registry entry for the entry point.
    Add its PROGRAM-ID to visited_programs.
    Add it as a node in call_graph.nodes.

  STEP 2 — BFS loop (process programs):
    WHILE queue is not empty:
      current_file = dequeue(queue)
      current_id   = PROGRAM-ID of current_file (uppercased)

      Run all grep patterns on current_file to find references.
      For each reference found:

        a. Normalise target name to UPPERCASE.

        b. If edge type is COPY or SQL_INCLUDE:
             Look up target in lookup_registry.copybooks_lookup.
             If found AND target NOT in visited_copybooks:
               Add copybook entry to copybook_registry.
               Add it as a node in call_graph.nodes.
               Add target to visited_copybooks.
             Add edge (current_id → target) to call_graph.edges.
             Add current_id to copybook_map[target].
             If not found in lookup: set resolved=false, log warning issue.

        c. If edge type is STATIC_CALL, CICS_LINK, or CICS_XCTL:
             Look up target in lookup_registry.programs_lookup.
             If found AND target NOT in visited_programs:
               Add program entry to file_registry.
               Add it as a node in call_graph.nodes.
               Add target to visited_programs.
               Enqueue the program's file path → queue.  ← triggers recursive scan
             Add edge (current_id → target) to call_graph.edges.
             If not found in lookup: set resolved=false, log warning issue.

        d. If edge type is DYNAMIC_CALL:
             Apply disambiguation rules.
             Add edge with target=null to call_graph.edges.
             Log info issue.

        e. If edge type is CICS_RETURN:
             Record the transaction ID in the edge — no file to enqueue.

  STEP 3 — JCL cross-reference (post-BFS):
    For each job in lookup_registry.jcl_lookup:
      If any step.program value matches a program ID in visited_programs:
        Add the JCL job entry to jcl_registry.
        Add it as a node in call_graph.nodes.
        Add an EXEC edge from the job to that program.

  STEP 4 — used_by population:
    For each (copybook_id, users_list) in copybook_map:
      Set copybook_registry[copybook_id].used_by = users_list.

  STEP 5 — Compute stats block from the collected lists.
```

**Key rule**: only enqueue a program file if it was found in
`lookup_registry.programs_lookup` AND has not been visited yet. This prevents
infinite loops on circular CALLs and avoids processing unreachable programs.

---

## Circular dependency handling

- Circular CALL chains (A calls B calls A): the visited_programs set prevents
  re-enqueuing — both will be in the output but the second edge is still
  recorded normally.
- Circular COPY chains (A copies B copies A): detect by checking
  `visited_copybooks` before adding — if B is already visited when processing
  A's COPY of B, record the edge but log a `circular_copy` error issue.

---

## Output specification

This skill produces the following sections of `inventory_artifact.json`:

### Full artifact schema

```json
{
  "meta": {
    "generated_at": "2024-01-15T10:30:00Z",
    "entry_point_file": "/absolute/path/to/MAINPROG.CBL",
    "entry_point_id": "MAINPROG",
    "search_root": "/absolute/path/to/search/folder/",
    "agent_version": "1_inventory@1.0",
    "total_programs_reachable": 12,
    "total_copybooks_reachable": 8
  },
  "stats": {
    "programs": <>,
    "batch_programs": <>,
    "online_programs": <>,
    "copybooks": <>,
    "jcl_jobs": <>,
    "bms_maps": <>,
    "db2_includes": <>,
    "call_edges_total": <>,
    "call_edges_resolved": <>,
    "call_edges_unresolved": <>,
    "dynamic_calls": <>,
    "copy_edges": <>,
    "cics_link_edges": <>,
    "cics_xctl_edges": <>
  },
  "file_registry": [
    {
      "id": "MAINPROG",
      "program_id": "MAINPROG",
      "path": "programs/batch/MAINPROG.CBL",
      "relative_path": "programs/batch/MAINPROG.CBL",
      "type": "program",
      "subtype": "batch",
      "size_bytes": 14200,
      "is_entry_point": true
    }
  ],
  "copybook_registry": [
    {
      "id": "CVACT01Y",
      "path": "copybook/common/CVACT01Y.CPY",
      "relative_path": "copybook/common/CVACT01Y.CPY",
      "type": "copybook",
      "used_by": ["MAINPROG", "SUBPROG1"]
    }
  ],
  "jcl_registry": [
    {
      "id": "ACCTFILE",
      "path": "jcl/ACCTFILE.JCL",
      "relative_path": "jcl/ACCTFILE.JCL",
      "job_name": "ACCTFILE",
      "steps": [
        { "step_name": "MAINSTEP", "program": "MAINPROG", "type": "cobol" }
      ]
    }
  ],
  "call_graph": {
    "nodes": [
      {
        "id": "MAINPROG",
        "type": "program",
        "subtype": "batch",
        "path": "programs/batch/MAINPROG.CBL",
        "is_entry_point": true
      },
      {
        "id": "CVACT01Y",
        "type": "copybook",
        "path": "copybook/common/CVACT01Y.CPY"
      }
    ],
    "edges": [
      {
        "from": "MAINPROG",
        "to": "SUBPROG1",
        "type": "STATIC_CALL",
        "resolved": true,
        "source_line_hint": 420
      },
      {
        "from": "MAINPROG",
        "to": "CVACT01Y",
        "type": "COPY",
        "resolved": true,
        "source_line_hint": 85
      },
      {
        "from": "MAINPROG",
        "to": null,
        "type": "DYNAMIC_CALL",
        "resolved": false,
        "variable": "WS-PROG-NAME",
        "confirmed": true,
        "source_line_hint": 530
      }
    ]
  },
  "copybook_map": {
    "CVACT01Y": ["MAINPROG", "SUBPROG1"],
    "CVCRD01Y": ["MAINPROG"]
  },
  "issues": [
    {
      "severity": "warning",
      "type": "unresolved_reference",
      "source": "MAINPROG",
      "reference": "EXTPROG1",
      "edge_type": "STATIC_CALL",
      "message": "CALL target 'EXTPROG1' not found in lookup registry — may be external or in load library"
    },
    {
      "severity": "info",
      "type": "dynamic_call",
      "source": "MAINPROG",
      "variable": "WS-PROG-NAME",
      "confirmed": true,
      "message": "Dynamic CALL via variable WS-PROG-NAME — cannot resolve statically"
    },
    {
      "severity": "warning",
      "type": "missing_program_id",
      "source_file": "programs/batch/OLDPROG.CBL",
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

Note: `is_entry_point: true` is set only on the root program supplied as
`ENTRY_POINT_FILE`. All other programs default to `is_entry_point: false`
(the field may be omitted for non-entry-point programs).

---

## Error handling

| Condition | Severity | Action |
|---|---|---|
| ENTRY_POINT_FILE not found on disk | `error` | Abort — report to agent |
| ENTRY_POINT_FILE not a `.cbl` or `.cob` file | `error` | Abort — report to agent |
| Target of CALL/COPY not in lookup_registry | `warning` | Add edge with `resolved: false`, log issue; do NOT enqueue |
| Dynamic CALL variable not confirmed in WS | `info` | Add edge with `confirmed: false`, log issue |
| Circular COPY chain | `error` | Log issue, do not re-process — record both parties |
| CICS LINK/XCTL target not in lookup_registry | `warning` | Add edge with `resolved: false`, log issue |
| Grep produces no output for a file | `info` | File has no outgoing references — normal for leaf programs |
| Program file found in lookup but cannot be read | `error` | Log issue, skip edge extraction for that file |
