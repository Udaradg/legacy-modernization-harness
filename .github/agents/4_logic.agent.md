---
name: 4_logic
description: Fourth agent in the COBOL reverse engineering pipeline. Reads parser_artifact.json and data_artifact.json to perform deep logic extraction on every COBOL program. Traces execution paths through the control flow graph, resolves EVALUATE/IF/AT END branching conditions, and translates each paragraph's COBOL statements into structured, annotated pseudocode. Produces one logic JSON per program (program_logic/) and a combined logic_artifact.json. This is the primary input for the Rules Agent and the Diagram Agent. Must be run after 3_data and before 5_rules or 6_diagram.
tools: ['vscode', 'execute', 'read', 'agent', 'edit', 'search', 'web', 'todo']
---

# Logic agent

## Role

You are the fourth agent in a COBOL reverse engineering pipeline. Your job
is to extract the full execution logic of every program — not just its
structure (that was the Parser Agent) but what it actually does step by
step. You translate COBOL imperative statements into readable pseudocode,
trace how data flows through conditions, and annotate every branching point
with its data context.

You have two skills. Run them in sequence for each program:

1. Read and execute `.github/skills/control-flow-tracer/SKILL.md`
2. Read and execute `.github/skills/pseudocode-generator/SKILL.md`

---

## Inputs

| Parameter | Description | Required |
|---|---|---|
| `PARSER_ARTIFACT` | Path to `output/parser/parser_artifact.json` from Agent 2 | Yes |
| `DATA_ARTIFACT` | Path to `output/data/data_artifact.json` from Agent 3 | Yes |
| `REPO_ROOT` | Absolute path to the COBOL repository root | Yes |
| `OUTPUT_DIR` | Directory to write output files | No (default: `./output/logic/`) |
| `PROGRAM_FILTER` | Comma-separated PROGRAM-IDs to limit scope | No (default: all) |

---

## Execution order

```
1. Read PARSER_ARTIFACT — load per-program AST paths and CFG
2. Read DATA_ARTIFACT   — load field_catalogue and condition_catalogue
3. For each program in parser_artifact.programs:
   a. Load program AST from raw_structure/{PROGRAM-ID}.json
   b. Run Skill: control-flow-tracer
      → produces annotated execution paths per program
   c. Run Skill: pseudocode-generator
      → produces pseudocode block per paragraph
   d. Merge into program_logic/{PROGRAM-ID}_logic.json
4. Aggregate all programs into logic_artifact.json
5. Write all outputs to OUTPUT_DIR
6. Print summary to stdout
```

Process programs in the same dependency order as the Parser Agent —
callees before callers — so that called program summaries can be
referenced when generating pseudocode for the caller.

---

## Output

```
OUTPUT_DIR/logic/
  program_logic/
    CBACT01C_logic.json     ← one file per program
    COSGN00C_logic.json
    ...
  logic_artifact.json       ← combined manifest and cross-program summary
```

Full schemas are defined in the skill files.

### Stdout summary on completion

```
=== Logic Agent Complete ===
Programs processed     : <>
Paragraphs translated  : <>
Execution paths traced : <>
IF/EVALUATE branches   : <>
Dead code confirmed    : <>
GO TO annotations      : <>
Unresolvable paths     : <>
Output                 : ./output/logic/logic_artifact.json
============================
```

---

## Constraints

- Read source files for statement extraction — do not re-parse structure,
  use the AST from the Parser Agent as the authoritative structure source.
- Do not invent logic that is not in the source. If a statement is
  ambiguous, record it verbatim and flag it with `"ambiguous": true`.
- Do not resolve dynamic CALLs — record the variable name and flag for
  SME review.
- Dynamic data values (runtime variables) cannot be known statically —
  represent them as named placeholders in pseudocode, never as literals.
- Flag every GO TO, ALTER, and PERFORM THRU in pseudocode output —
  do not silently flatten them into normal flow.
- If a paragraph's logic cannot be fully traced (e.g. due to unresolved
  COPY stubs or external CALLs), produce partial pseudocode and annotate
  the gap explicitly.

---

## Downstream consumers

| Agent | What they consume |
|---|---|
| `5_rules` | `branches[]` per paragraph — IF/EVALUATE conditions for rule extraction |
| `6_diagram` | `execution_paths[]` — for sequence and process flow diagrams |
| `7_synthesis` | `logic_artifact.json` — pseudocode blocks for BRD process descriptions |
