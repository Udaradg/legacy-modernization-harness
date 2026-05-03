---
name: section-mapper
description: >
  Takes the AST JSON produced by the cobol-ast-parser skill for a single
  program and enriches it with a control flow graph. Scans the PROCEDURE
  DIVISION of each paragraph for PERFORM, PERFORM UNTIL, PERFORM VARYING,
  PERFORM THRU, GO TO, and ALTER statements. Resolves each to the target
  paragraph or section name. Builds a directed control flow graph where
  nodes are paragraphs and edges are typed control transfers. Flags
  unstructured constructs (GO TO, ALTER, PERFORM THRU) for the Logic Agent.
  Called once per program by the Parser Agent, after cobol-ast-parser.
---

## Purpose

Read the paragraph boundaries from the AST produced by cobol-ast-parser.
For each paragraph, scan its source lines and detect all control transfer
statements: PERFORM variants, GO TO, and ALTER. Build a typed, directed
control flow graph (CFG) at the paragraph level.

Do NOT interpret what the control flow means for business logic. Record
structure only — who calls whom, what type of call, and any flags that
the Logic Agent needs to investigate.

---

## Control transfer statement types

| Statement | Edge type | Notes |
|---|---|---|
| `PERFORM para-name` | `PERFORM_SIMPLE` | Single paragraph call, returns |
| `PERFORM para-name THRU para-name-2` | `PERFORM_THRU` | Range of paragraphs — flag as complex |
| `PERFORM para-name UNTIL condition` | `PERFORM_UNTIL` | Loop — record condition text verbatim |
| `PERFORM para-name VARYING ... UNTIL` | `PERFORM_VARYING` | Indexed loop — record index and bounds |
| `PERFORM para-name TIMES` | `PERFORM_TIMES` | Fixed-count loop |
| `PERFORM section-name` | `PERFORM_SECTION` | Entire section call |
| `GO TO para-name` | `GOTO` | Unconditional jump — flag as unstructured |
| `GO TO para-1 para-2 ... DEPENDING ON var` | `GOTO_DEPENDING` | Computed jump — flag, record variable name |
| `ALTER para-name TO PROCEED TO para-name-2` | `ALTER` | Modifies a GO TO target — flag as critical |
| `NEXT SENTENCE` | `NEXT_SENTENCE` | Falls to next sentence — record position |
| `EXIT` | `EXIT` | Paragraph exit — no transfer |

---

## Grep patterns per paragraph

For each paragraph, extract its source lines (start_line to end_line from AST).
Then apply these patterns within those lines only:

```bash
# Extract just the paragraph's lines from the file
sed -n "${START},${END}p" "$FILE" > /tmp/para_lines.txt

# PERFORM simple
grep -niE "PERFORM\s+([A-Z0-9\-]+)(\s+THRU\s+([A-Z0-9\-]+))?" /tmp/para_lines.txt

# PERFORM UNTIL
grep -niE "PERFORM\s+([A-Z0-9\-]+)\s+UNTIL\s+(.*)" /tmp/para_lines.txt

# PERFORM VARYING
grep -niE "PERFORM\s+([A-Z0-9\-]+)\s+VARYING\s+(.*)" /tmp/para_lines.txt

# PERFORM TIMES
grep -niE "PERFORM\s+([A-Z0-9\-]+)\s+([0-9]+|[A-Z0-9\-]+)\s+TIMES" /tmp/para_lines.txt

# Inline PERFORM (no target — body follows)
grep -niE "PERFORM\s+(UNTIL|VARYING|WITH\s+TEST)" /tmp/para_lines.txt

# GO TO simple
grep -niE "GO\s+TO\s+([A-Z0-9\-]+)" /tmp/para_lines.txt

# GO TO DEPENDING ON
grep -niE "GO\s+TO\s+(([A-Z0-9\-]+\s+)+)DEPENDING\s+ON\s+([A-Z0-9\-]+)" /tmp/para_lines.txt

# ALTER
grep -niE "ALTER\s+([A-Z0-9\-]+)\s+TO\s+(PROCEED\s+TO\s+)?([A-Z0-9\-]+)" /tmp/para_lines.txt
```

---

## PERFORM THRU handling

`PERFORM A-PARA THRU B-PARA` means all paragraphs between A-PARA and B-PARA
(in source order) are executed sequentially.

When detected:
1. Record a `PERFORM_THRU` edge from source paragraph to A-PARA
2. Look up the paragraph list from the AST to find all paragraphs between
   A-PARA and B-PARA (inclusive) in source order
3. Record an `implicit_thru_range` array listing all paragraphs in the range
4. Flag the edge with `"unstructured": true`
5. Log a `warning` issue — PERFORM THRU obscures control flow

---

## Inline PERFORM handling

`PERFORM UNTIL ... END-PERFORM` (no target paragraph) is an inline loop.
The body is contained within the same paragraph.

When detected:
1. Create a self-referencing node: edge from paragraph back to itself
2. Edge type: `PERFORM_INLINE`
3. Record `"inline": true` and the condition text verbatim
4. Record start line and `END-PERFORM` line as the loop boundary

```bash
# Find END-PERFORM to close inline loop
grep -niE "END-PERFORM" /tmp/para_lines.txt
```

---

## Target resolution

For each detected PERFORM or GO TO target:
1. Look up the target name in the AST's `procedure_division.paragraphs` list
2. If found → `"resolved": true`
3. If not found in paragraphs → look in `procedure_division.sections`
4. If still not found → `"resolved": false`, log `warning`
   (may be in a COPYed section or external program)

---

## Unstructured construct flags

Flag the following with `"unstructured": true` on the edge AND add an
entry to the program's `unstructured_constructs` list:

- Any `GO TO` edge
- Any `GO TO ... DEPENDING ON` edge
- Any `PERFORM THRU` edge
- Any `ALTER` statement (flag as `"severity": "critical"` — ALTER is the
  most dangerous COBOL construct; it changes GO TO targets at runtime)

These flags are consumed by the Logic Agent and the Synthesis Agent
(BRD gap section).

---

## Control flow graph construction rules

```
1. Start with the paragraph list from the AST as nodes
2. For each paragraph:
   a. Scan its line range using the grep patterns above
   b. For each match create an edge (see schema below)
   c. Resolve the target to a known paragraph or section
   d. Apply unstructured flags where needed
3. Identify entry points:
   - The first paragraph in PROCEDURE DIVISION is the implicit entry point
   - Any paragraph named in a JCL EXEC step is also an entry point
   - Any paragraph targeted by CICS LINK or XCTL is also an entry point
4. Identify terminal nodes:
   - Paragraphs containing STOP RUN, GOBACK, or EXIT PROGRAM
5. Identify dead paragraphs:
   - Paragraphs with no incoming edges and not an entry point → flag as dead code
```

---

## Output schema — control flow graph (added to program AST)

This skill adds a `control_flow_graph` section to the program AST JSON
produced by cobol-ast-parser:

```json
{
  "control_flow_graph": {
    "entry_points": ["0000-MAIN"],
    "terminal_nodes": ["9999-EXIT"],
    "nodes": [
      {
        "id": "0000-MAIN",
        "type": "paragraph",
        "section": "0000-MAIN",
        "start_line": 425,
        "end_line": 480,
        "is_entry_point": true,
        "is_terminal": false,
        "is_dead_code": false,
        "unstructured_constructs": []
      },
      {
        "id": "0100-OPEN-FILES",
        "type": "paragraph",
        "section": "0000-MAIN",
        "start_line": 482,
        "end_line": 510,
        "is_entry_point": false,
        "is_terminal": false,
        "is_dead_code": false,
        "unstructured_constructs": []
      },
      {
        "id": "9999-EXIT",
        "type": "paragraph",
        "section": null,
        "start_line": 885,
        "end_line": 892,
        "is_entry_point": false,
        "is_terminal": true,
        "is_dead_code": false,
        "unstructured_constructs": []
      }
    ],
    "edges": [
      {
        "from": "0000-MAIN",
        "to": "0100-OPEN-FILES",
        "type": "PERFORM_SIMPLE",
        "resolved": true,
        "unstructured": false,
        "source_line": 431
      },
      {
        "from": "0000-MAIN",
        "to": "0200-PROCESS-RECORDS",
        "type": "PERFORM_UNTIL",
        "resolved": true,
        "unstructured": false,
        "condition_text": "WS-EOF-FLAG = 'Y'",
        "source_line": 445
      },
      {
        "from": "0300-VALIDATE",
        "to": "0350-ERROR-HANDLER",
        "type": "GOTO",
        "resolved": true,
        "unstructured": true,
        "source_line": 612
      },
      {
        "from": "0400-CALC-LOOP",
        "to": "0400-CALC-LOOP",
        "type": "PERFORM_INLINE",
        "resolved": true,
        "unstructured": false,
        "inline": true,
        "condition_text": "WS-IDX > WS-MAX-ITEMS",
        "loop_start_line": 700,
        "loop_end_line": 748,
        "source_line": 700
      },
      {
        "from": "0500-ROUTE",
        "to": null,
        "type": "GOTO_DEPENDING",
        "resolved": false,
        "unstructured": true,
        "depending_variable": "WS-TRANS-TYPE",
        "possible_targets": ["1000-CREDIT", "1100-DEBIT", "1200-TRANSFER"],
        "source_line": 820
      }
    ],
    "unstructured_constructs": [
      {
        "type": "GOTO",
        "paragraph": "0300-VALIDATE",
        "target": "0350-ERROR-HANDLER",
        "line": 612,
        "severity": "warning"
      },
      {
        "type": "GOTO_DEPENDING",
        "paragraph": "0500-ROUTE",
        "depending_variable": "WS-TRANS-TYPE",
        "line": 820,
        "severity": "warning"
      }
    ],
    "dead_code_candidates": [],
    "cfg_issues": []
  }
}
```

---

## Parser artifact manifest

After all programs are processed, the Parser Agent writes a combined
`parser_artifact.json`:

```json
{
  "meta": {
    "generated_at": "ISO-8601 timestamp",
    "agent_version": "2_parser@1.0",
    "source_artifact": "inventory_artifact.json",
    "programs_parsed": 42,
    "programs_failed": 2
  },
  "stats": {
    "total_paragraphs": 1847,
    "total_sections": 203,
    "total_perform_edges": 634,
    "total_goto_edges": 89,
    "total_alter_statements": 3,
    "total_inline_performs": 47,
    "total_dead_code_candidates": 12,
    "programs_with_unstructured_flow": 18
  },
  "programs": [
    {
      "program_id": "CBACT01C",
      "source_file": "app/cbl/CBACT01C.CBL",
      "ast_file": "raw_structure/CBACT01C.json",
      "total_lines": 892,
      "paragraph_count": 24,
      "section_count": 5,
      "has_unstructured_flow": true,
      "has_alter": false,
      "dead_code_candidates": 0,
      "parse_status": "success"
    }
  ],
  "issues": []
}
```

---

## Error handling

| Condition | Action |
|---|---|
| Paragraph target not found in AST | Log `warning` in `cfg_issues`, set `resolved: false` |
| ALTER statement detected | Log `critical` issue — record both the ALTER paragraph and target |
| PERFORM THRU range is ambiguous | Log `warning`, record start only, set `implicit_thru_range: null` |
| Inline PERFORM missing END-PERFORM | Log `error`, mark loop boundary as end of paragraph |
| Paragraph with no outgoing edges (not terminal) | Flag as `dead_code_candidate` with `info` severity |
| GO TO targeting a paragraph in a different section | Log `warning` — cross-section jump increases complexity |
