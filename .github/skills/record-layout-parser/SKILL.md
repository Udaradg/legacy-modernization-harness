---
name: record-layout-parser
description: >
  Parses COBOL copybook files and inline WORKING-STORAGE sections to extract
  the full field-level layout of every data record. Resolves level number
  hierarchies, expands PIC clauses into typed field definitions, handles
  REDEFINES aliases, OCCURS tables, and 88-level condition names. Produces
  one structured JSON layout file per copybook and per program WS section.
  COPY stubs identified by the Parser Agent are expanded using the parsed
  copybook layouts. Called by the Data Agent before data-dictionary-builder.
---

# Skill — record layout parser

## Purpose

Open each copybook file and each program's inline WORKING-STORAGE section.
Walk the level-number hierarchy top-down. For every field, extract its name,
level, PIC clause (parsed into type/length/decimal), REDEFINES target,
OCCURS clause, VALUE, and 88-level conditions. Produce a flat-then-nested
JSON layout that the data-dictionary-builder can normalise.

Do NOT interpret what fields mean. Record structure and attributes only.

---

## COBOL fixed-format column rules

Apply to all `.cbl` and `.cpy` files:

| Columns (1-based) | Action |
|---|---|
| 1–6 | Ignore (sequence numbers) |
| 7 | `*` or `/` → skip line (comment) |
| 8–11 | Area A — 01, 77-level entries start here |
| 12–72 | Area B — subordinate entries and clauses start here |
| 73–80 | Ignore |

Continuation lines: if column 7 is `-`, the line continues the previous
line's clause. Concatenate columns 12–72 of the continuation onto the
previous line before parsing.

---

## Phase 1 — level number hierarchy extraction

### Detect all data entries

```bash
# All level-numbered entries (01–49, 77, 88) — skip comment lines
grep -niE "^.{6}[^*/]\s+(0?1|0?2|0?3|0?4|0?5|0?6|0?7|0?8|0?9|[1-4][0-9]|77|88)\s+([A-Z0-9\-]+)" "$FILE"
```

For each match record:
- Line number
- Level number (normalise: strip leading zero → integer)
- Data name (if `FILLER` → generate `FILLER_NNNN` using line number)
- Raw remainder of the line (contains PIC, OCCURS, REDEFINES, VALUE clauses)

### Hierarchy rules

Build a tree from the flat list using these rules:

```
- Level 01 and 77 → root entries (direct children of their section)
- Level 02–49 → child of the nearest preceding entry with a lower level number
- Level 88 → condition name, always child of its immediately preceding entry
  (regardless of level number difference)
- Level 66 → RENAMES entry — record separately, do not nest as child
- If level sequence skips (e.g. 01 → 05 → 03) → log warning, attach 03
  as sibling of 05 (same parent as 05)
```

---

## Phase 2 — PIC clause parsing

For each elementary item (an entry that has a PIC clause), extract and
parse the PIC string:

```bash
# PIC clause — may use PIC or PICTURE, with optional IS
grep -iE "PIC(TURE)?\s+(IS\s+)?([X9AVSZB\(\)\/\+\-\.\,\*\$]+)" <<< "$REMAINDER"
```

### PIC clause type classification

| PIC pattern | COBOL type | Normalised type | Notes |
|---|---|---|---|
| `9(n)` | Numeric | `INTEGER` | Length = n |
| `9(n)V9(d)` | Numeric decimal | `DECIMAL` | Length = n+d, decimal = d |
| `S9(n)` | Signed numeric | `SIGNED_INTEGER` | Sign flag = true |
| `S9(n)V9(d)` | Signed decimal | `SIGNED_DECIMAL` | Sign flag = true |
| `9(n) COMP` / `9(n) COMP-3` | Binary/packed | `BINARY` / `PACKED_DECIMAL` | Note storage type |
| `X(n)` | Alphanumeric | `STRING` | Length = n |
| `A(n)` | Alphabetic | `ALPHA` | Length = n |
| `X` (no parens) | Alphanumeric | `STRING` | Length = 1 |
| `9` (no parens) | Numeric | `INTEGER` | Length = 1 |
| `V9(n)` | Implied decimal | `DECIMAL` | Length = n, integer part = 0 |
| `9(n)V` | Integer with implied decimal | `DECIMAL` | Decimal = 0 |
| `AAAA...` (repeated) | Alphabetic | `ALPHA` | Count repeated chars |
| `XXXX...` (repeated) | Alphanumeric | `STRING` | Count repeated chars |
| `9999...` (repeated) | Numeric | `INTEGER` | Count repeated chars |

### PIC length extraction

For PIC `X(15)` → length = 15
For PIC `XXX` → count characters → length = 3
For PIC `9(7)V99` → integer digits = 7, decimal digits = 2, total length = 9
For PIC `S9(5) COMP-3` → packed decimal, length = 3 bytes (ceiling of (5+1)/2)

Record both the raw PIC string and the parsed attributes.

### USAGE clause (storage type)

```bash
grep -iE "USAGE\s+(IS\s+)?(COMP|COMP-1|COMP-2|COMP-3|COMP-4|COMP-5|BINARY|PACKED-DECIMAL|DISPLAY|INDEX|POINTER)" <<< "$REMAINDER"
```

| USAGE value | Storage type |
|---|---|
| `DISPLAY` (default) | Standard character storage |
| `COMP` / `COMP-4` / `BINARY` | Binary integer |
| `COMP-1` | Single-precision float |
| `COMP-2` | Double-precision float |
| `COMP-3` / `PACKED-DECIMAL` | Packed decimal (BCD) |
| `COMP-5` | Native binary |
| `INDEX` | Table index |
| `POINTER` | Memory pointer |

---

## Phase 3 — REDEFINES extraction

```bash
grep -iE "REDEFINES\s+([A-Z0-9\-]+)" <<< "$REMAINDER"
```

Rules:
- Record the target field name (what is being redefined)
- The REDEFINES entry occupies the same storage as the target
- Both entries should have the same total byte length — if not, log `warning`
- Do NOT follow the REDEFINES to resolve target structure here; the
  data-dictionary-builder resolves the alias relationship
- Multiple entries can REDEFINES the same target — record all of them

---

## Phase 4 — OCCURS extraction

```bash
# Simple OCCURS
grep -iE "OCCURS\s+([0-9]+)\s+TIMES?" <<< "$REMAINDER"

# OCCURS with variable limit (ODO — OCCURS DEPENDING ON)
grep -iE "OCCURS\s+([0-9]+)\s+TO\s+([0-9]+)\s+TIMES?\s+DEPENDING\s+ON\s+([A-Z0-9\-]+)" <<< "$REMAINDER"

# INDEXED BY
grep -iE "INDEXED\s+BY\s+([A-Z0-9\-]+)" <<< "$REMAINDER"

# KEY IS (for sorted tables)
grep -iE "(ASCENDING|DESCENDING)\s+KEY\s+(IS\s+)?([A-Z0-9\-]+)" <<< "$REMAINDER"
```

Record:
- Minimum occurs (0 if DEPENDING ON, else same as max)
- Maximum occurs
- Whether variable-length (DEPENDING ON)
- Depending-on field name if variable-length
- Index name(s) from INDEXED BY
- Key field(s) and sort order if present

---

## Phase 5 — 88-level condition extraction

```bash
# 88-level conditions — VALUE or VALUES, single or list
grep -niE "^.{6}[^*/]\s+88\s+([A-Z0-9\-]+)\s+(VALUE(S)?\s+(IS|ARE)?\s+(.+))" "$FILE"
```

For each 88-level entry record:
- Condition name
- Parent field name (the immediately preceding non-88 entry)
- Value list (may be a single value, a list, or a THRU range)
- Whether it is a negation (FALSE clause)

Value list patterns:
- Single: `VALUE 'Y'` → `["Y"]`
- List: `VALUES 'A' 'B' 'C'` → `["A", "B", "C"]`
- Range: `VALUE 1 THRU 9` → `{"from": 1, "to": 9}`
- Mixed: `VALUES 0 THRU 5 10 20` → `[{"from": 0, "to": 5}, 10, 20]`

---

## Phase 6 — COPY stub expansion

After parsing all copybooks, go back to each program's AST and expand
COPY stubs recorded by the cobol-ast-parser skill:

```
For each copy_stub in program AST:
  1. Look up copybook_id in parsed copybook layouts
  2. If found: inline the copybook's field tree at the stub's position
     - Inherit the parent level context (e.g. if stub is inside a level 01,
       adjust child level numbers accordingly — REPLACING clause may apply)
  3. If REPLACING clause present: apply text substitution to all field names
     before inlining
  4. If not found: leave as unresolved stub, log warning
```

### COPY REPLACING handling

```bash
# COPY with REPLACING clause
grep -iE "COPY\s+([A-Z0-9\-]+)\s+REPLACING\s+(.*)" "$FILE"
```

REPLACING syntax: `==OLD-TEXT== BY ==NEW-TEXT==`
Apply all substitution pairs to field names and literals in the copybook
before inlining. Record the substitutions in the layout output.

---

## Output schema — layout file per copybook/program WS

```json
{
  "id": "CVACT01Y",
  "source_type": "copybook",
  "source_file": "app/copy/CVACT01Y.CPY",
  "used_by_programs": ["CBACT01C", "COACT01C", "COACT02C"],
  "total_fields": 24,
  "total_bytes": 300,
  "records": [
    {
      "level": 1,
      "name": "ACCOUNT-RECORD",
      "pic": null,
      "type": "GROUP",
      "storage_type": "DISPLAY",
      "bytes": 300,
      "redefines": null,
      "occurs": null,
      "value": null,
      "line": 5,
      "children": [
        {
          "level": 5,
          "name": "ACCT-ID",
          "pic": "X(11)",
          "pic_parsed": {
            "raw": "X(11)",
            "normalized_type": "STRING",
            "length": 11,
            "decimal_places": 0,
            "signed": false,
            "storage_type": "DISPLAY"
          },
          "bytes": 11,
          "redefines": null,
          "occurs": null,
          "value": null,
          "line": 7,
          "children": []
        },
        {
          "level": 5,
          "name": "ACCT-BALANCE",
          "pic": "S9(10)V99 COMP-3",
          "pic_parsed": {
            "raw": "S9(10)V99 COMP-3",
            "normalized_type": "SIGNED_DECIMAL",
            "length": 12,
            "decimal_places": 2,
            "signed": true,
            "storage_type": "PACKED_DECIMAL",
            "bytes": 7
          },
          "redefines": null,
          "occurs": null,
          "value": "ZEROS",
          "line": 9,
          "children": [
            {
              "level": 88,
              "name": "ACCT-OVERDRAWN",
              "pic": null,
              "type": "CONDITION",
              "parent_field": "ACCT-BALANCE",
              "values": [{"from": -999999999.99, "to": -0.01}],
              "line": 10,
              "children": []
            }
          ]
        },
        {
          "level": 5,
          "name": "FILLER_0015",
          "pic": "X(10)",
          "pic_parsed": {
            "raw": "X(10)",
            "normalized_type": "STRING",
            "length": 10,
            "signed": false,
            "storage_type": "DISPLAY"
          },
          "is_filler": true,
          "line": 15,
          "children": []
        },
        {
          "level": 5,
          "name": "ACCT-TYPE-REDEF",
          "pic": "9(3)",
          "redefines": "ACCT-TYPE-CODE",
          "pic_parsed": {
            "raw": "9(3)",
            "normalized_type": "INTEGER",
            "length": 3
          },
          "line": 22,
          "children": []
        },
        {
          "level": 5,
          "name": "TRANS-TABLE",
          "pic": null,
          "type": "GROUP",
          "occurs": {
            "min": 1,
            "max": 12,
            "variable_length": false,
            "depending_on": null,
            "indexed_by": ["TRANS-IDX"],
            "keys": [{"field": "TRANS-DATE", "order": "ASCENDING"}]
          },
          "line": 28,
          "children": []
        }
      ]
    }
  ],
  "layout_issues": []
}
```

---

## Error handling

| Condition | Action |
|---|---|
| PIC clause spans multiple lines | Concatenate continuation lines before parsing |
| PIC clause is unrecognised | Record raw string, set `normalized_type: "UNKNOWN"`, log `warning` |
| REDEFINES target not found in same record | Log `warning`, record `redefines_resolved: false` |
| OCCURS DEPENDING ON variable not in same record | Log `warning`, record field name only |
| Copybook not found for COPY stub | Leave as unresolved stub, log `warning` |
| REPLACING clause substitution produces duplicate name | Log `warning`, append `_R` suffix |
| Level sequence invalid | Log `warning`, best-effort parent assignment |
| Continuation line missing | Log `warning`, parse what is available |
