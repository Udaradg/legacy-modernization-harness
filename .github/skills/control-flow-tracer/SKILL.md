---
name: control-flow-tracer
description: >
  Takes a single program's AST (from the Parser Agent) and its control flow
  graph, then walks all execution paths from each entry point. Resolves
  PERFORM chains into ordered call sequences, traces IF/EVALUATE/AT END
  branching conditions, annotates each branch with the data fields it tests,
  identifies loops and their termination conditions, confirms or refutes
  dead code candidates, and produces an annotated execution trace with
  path summaries. Called by the Logic Agent before pseudocode-generator.
---

# Skill — control flow tracer

## Purpose

Starting from each entry point paragraph, walk the control flow graph
produced by the Parser Agent. Follow every PERFORM edge, resolve branch
conditions by reading the actual COBOL source lines, and build an ordered
execution trace that shows what runs in what sequence, under what conditions.

Annotate every branching point with the fields it tests and the outcomes
of each branch. Identify loops and their termination conditions. Confirm
or dismiss dead code candidates flagged by the Parser Agent.

Do NOT translate statements to pseudocode — that is the pseudocode-generator
skill's job. Extract execution order and branch conditions only.

---

## Phase 1 — entry point identification

Load entry points from the program AST's `control_flow_graph.entry_points`.

Supplement with:
- The first paragraph in `procedure_division.paragraphs` (always an entry)
- Any paragraph referenced in `jcl_registry` steps (batch entry points)
- Any paragraph targeted by `CICS_LINK` or `CICS_XCTL` edges in the
  inventory call graph (online entry points)

For each entry point, initialise a separate execution trace.

---

## Phase 2 — PERFORM chain resolution

Walk the CFG edge list from the current paragraph. For each outgoing edge:

### PERFORM_SIMPLE
```
Record: call to target paragraph
Next: after target paragraph returns, continue from calling paragraph
```

### PERFORM_UNTIL
```bash
# Read the condition from source lines around the PERFORM statement
sed -n "${PERFORM_LINE}p" "$SOURCE_FILE" | grep -iE "UNTIL\s+(.*)"
```
Record:
- Target paragraph name
- Condition text verbatim (raw COBOL)
- Fields referenced in condition (look up each token in field_catalogue)
- Condition test time: `BEFORE` (default) or `AFTER` (WITH TEST AFTER)
- Loop type: `WHILE` (test before) or `DO_WHILE` (test after)

### PERFORM_VARYING
```bash
sed -n "${PERFORM_LINE},${PERFORM_LINE+5}p" "$SOURCE_FILE" | \
  grep -iE "VARYING\s+([A-Z0-9\-]+)\s+FROM\s+(.*)\s+BY\s+(.*)\s+UNTIL\s+(.*)"
```
Record:
- Index variable name
- FROM value (literal or field name)
- BY value (increment — literal or field name)
- UNTIL condition text
- Fields referenced

### PERFORM_THRU
Record the full paragraph range (from AST `implicit_thru_range`).
Flag as `"unstructured": true`.
Add each paragraph in range as an ordered sub-step in the trace.

### PERFORM_INLINE
```bash
# Find END-PERFORM boundary
sed -n "${LOOP_START},${PARA_END}p" "$SOURCE_FILE" | grep -niE "END-PERFORM"
```
Record as a self-contained block with condition and body line range.

---

## Phase 3 — branch condition extraction

For every branching statement inside each paragraph, extract the full
condition and map it to data fields.

### IF statement extraction

```bash
# IF statement — may span multiple lines
sed -n "${START},${END}p" "$SOURCE_FILE" | grep -niE "^\s{0,6}[^*/].{4,}\bIF\b\s+(.*)"
```

For each IF, trace through to find matching ELSE and END-IF:

```bash
# ELSE clause
grep -niE "^\s{0,6}[^*/].{4,}\bELSE\b" /tmp/para_lines.txt

# END-IF
grep -niE "^\s{0,6}[^*/].{4,}\bEND-IF\b" /tmp/para_lines.txt
```

Record:
```json
{
  "type": "IF",
  "line": 445,
  "condition_text": "WS-RETURN-CODE NOT EQUAL ZEROS",
  "condition_fields": ["WS-RETURN-CODE"],
  "has_else": true,
  "then_line_range": [446, 452],
  "else_line_range": [454, 460],
  "end_if_line": 461,
  "nested_depth": 1
}
```

Detect nested IFs by tracking depth counter — increment on `IF`, decrement
on `END-IF`. Flag nesting depth > 3 as `"complex": true`.

### EVALUATE statement extraction

```bash
# EVALUATE header
grep -niE "^\s{0,6}[^*/].{4,}\bEVALUATE\b\s+(.*)" /tmp/para_lines.txt

# WHEN clauses
grep -niE "^\s{0,6}[^*/].{4,}\bWHEN\b\s+(.*)" /tmp/para_lines.txt

# WHEN OTHER
grep -niE "^\s{0,6}[^*/].{4,}\bWHEN\s+OTHER\b" /tmp/para_lines.txt

# END-EVALUATE
grep -niE "^\s{0,6}[^*/].{4,}\bEND-EVALUATE\b" /tmp/para_lines.txt
```

For EVALUATE TRUE ALSO TRUE (multi-subject):
- Record each subject separately
- Each WHEN clause has multiple conditions (one per subject)
- Record as a condition matrix

```json
{
  "type": "EVALUATE",
  "line": 520,
  "subjects": ["TRUE", "TRUE"],
  "when_clauses": [
    {
      "conditions": ["WS-TRANS-TYPE = 'CR'", "WS-AMOUNT > ZERO"],
      "condition_fields": ["WS-TRANS-TYPE", "WS-AMOUNT"],
      "body_line_range": [522, 528]
    },
    {
      "conditions": ["WS-TRANS-TYPE = 'DR'", "ANY"],
      "condition_fields": ["WS-TRANS-TYPE"],
      "body_line_range": [530, 536]
    },
    {
      "conditions": ["OTHER"],
      "condition_fields": [],
      "body_line_range": [538, 542],
      "is_default": true
    }
  ],
  "end_evaluate_line": 543
}
```

### AT END / INVALID KEY / ON EXCEPTION extraction

These are implicit branches on I/O operations:

```bash
# READ ... AT END
grep -niE "\bREAD\b.+\bAT\s+END\b" /tmp/para_lines.txt

# READ ... NOT AT END
grep -niE "\bREAD\b.+\bNOT\s+AT\s+END\b" /tmp/para_lines.txt

# WRITE/REWRITE ... INVALID KEY
grep -niE "\b(WRITE|REWRITE|DELETE)\b.+\bINVALID\s+KEY\b" /tmp/para_lines.txt

# CALL ... ON EXCEPTION / ON OVERFLOW
grep -niE "\bCALL\b.+\b(ON\s+EXCEPTION|ON\s+OVERFLOW)\b" /tmp/para_lines.txt
```

Record:
```json
{
  "type": "AT_END",
  "operation": "READ",
  "file_name": "ACCT-FILE",
  "line": 612,
  "at_end_line_range": [613, 616],
  "not_at_end_line_range": [618, 625],
  "condition_fields": ["ACCT-FILE"],
  "io_status_field": "WS-FILE-STATUS"
}
```

---

## Phase 4 — loop detection and termination analysis

For every PERFORM_UNTIL, PERFORM_VARYING, and PERFORM_INLINE edge in the CFG:

1. Extract the termination condition fields
2. Look up each field in `data_artifact.field_catalogue`
3. Identify what statements inside the loop body modify those fields
4. Determine loop termination pattern:

| Pattern | Classification |
|---|---|
| Field set by READ AT END | `FILE_EXHAUSTION_LOOP` |
| Field set by counter increment vs fixed limit | `COUNTED_LOOP` |
| Field set by external CALL result | `EXTERNAL_TERMINATION_LOOP` |
| Field never modified in loop body | `POTENTIAL_INFINITE_LOOP` — flag warning |
| Field modified conditionally | `CONDITIONAL_TERMINATION_LOOP` |

```bash
# Find all MOVE statements that set the termination condition field
grep -niE "\bMOVE\b.+\bTO\s+${FIELD_NAME}\b" /tmp/para_lines.txt

# Find all COMPUTE/ADD/SUBTRACT statements
grep -niE "\b(COMPUTE|ADD|SUBTRACT)\b.*\b${FIELD_NAME}\b" /tmp/para_lines.txt
```

---

## Phase 5 — dead code confirmation

The Parser Agent flagged dead code candidates (paragraphs with no incoming
edges and not an entry point). Confirm or dismiss each:

```
For each dead_code_candidate paragraph:
  1. Search ALL program source lines for the paragraph name as a PERFORM target
     grep -niE "PERFORM\s+${PARA_NAME}" "$SOURCE_FILE"
  2. Search for it as a GO TO target
     grep -niE "GO\s+TO\s+${PARA_NAME}" "$SOURCE_FILE"
  3. Search for it in PERFORM THRU ranges
     grep -niE "THRU\s+${PARA_NAME}" "$SOURCE_FILE"
  4. If any match found → dismiss dead code flag (was missed by CFG builder)
  5. If no match found → confirm dead code, record in trace
```

---

## Output schema — execution trace (added to program logic file)

```json
{
  "program_id": "CBACT01C",
  "execution_traces": [
    {
      "entry_point": "0000-MAIN",
      "entry_type": "batch_main",
      "steps": [
        {
          "step": 1,
          "paragraph": "0000-MAIN",
          "action": "PERFORM",
          "target": "0100-OPEN-FILES",
          "condition": null,
          "line": 431
        },
        {
          "step": 2,
          "paragraph": "0000-MAIN",
          "action": "PERFORM_UNTIL",
          "target": "0200-PROCESS-RECORDS",
          "condition": {
            "text": "WS-EOF-FLAG = 'Y'",
            "fields": ["WS-EOF-FLAG"],
            "test_time": "BEFORE",
            "loop_type": "WHILE",
            "termination_pattern": "FILE_EXHAUSTION_LOOP"
          },
          "line": 445
        },
        {
          "step": 3,
          "paragraph": "0000-MAIN",
          "action": "PERFORM",
          "target": "9000-CLOSE-FILES",
          "condition": null,
          "line": 460
        },
        {
          "step": 4,
          "paragraph": "0000-MAIN",
          "action": "STOP_RUN",
          "target": null,
          "condition": null,
          "line": 465
        }
      ]
    }
  ],
  "branch_map": [
    {
      "paragraph": "0200-PROCESS-RECORDS",
      "branches": [
        {
          "type": "IF",
          "line": 520,
          "condition_text": "WS-ACCT-TYPE = 'CREDIT'",
          "condition_fields": ["WS-ACCT-TYPE"],
          "outcomes": [
            { "label": "THEN", "line_range": [521, 530] },
            { "label": "ELSE", "line_range": [532, 540] }
          ]
        }
      ]
    }
  ],
  "loops": [
    {
      "paragraph": "0000-MAIN",
      "target": "0200-PROCESS-RECORDS",
      "type": "PERFORM_UNTIL",
      "condition_fields": ["WS-EOF-FLAG"],
      "termination_pattern": "FILE_EXHAUSTION_LOOP",
      "line": 445
    }
  ],
  "dead_code_confirmed": [],
  "unresolvable_paths": [],
  "trace_issues": []
}
```

---

## Error handling

| Condition | Action |
|---|---|
| Entry point paragraph not found in AST | Log `error`, skip trace for that entry point |
| PERFORM target not in CFG | Log `warning`, record as `"unresolvable"` step |
| IF without matching END-IF | Log `warning`, estimate boundary at next paragraph start |
| EVALUATE without END-EVALUATE | Log `warning`, treat next WHEN or paragraph boundary as end |
| Loop with no termination condition field found | Log `warning`, classify as `POTENTIAL_INFINITE_LOOP` |
| Nested IF depth > 5 | Log `warning`, flag as `"high_complexity"` |
| GO TO mid-paragraph | Log in `trace_issues`, record as unconditional jump step |
| Circular PERFORM chain (recursion) | Log `error`, record cycle and stop tracing that path |
