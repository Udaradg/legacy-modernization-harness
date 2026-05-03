---
name: cobol-ast-parser
description: >
  Opens a single COBOL program file and extracts its full structural
  hierarchy: all four DIVISIONs, SECTIONs within each, paragraph names
  and their line boundaries in the PROCEDURE DIVISION, WORKING-STORAGE
  entries (01-level groups and their children), FILE SECTION FD/SD
  descriptors, LINKAGE SECTION entries, and COPY statement positions.
  Produces a structured AST-like JSON object for one program. Called once
  per program by the Parser Agent. Does not interpret logic or data meaning.
---

## Purpose

Read one COBOL source file top to bottom. Identify and record every
structural boundary — where each DIVISION starts, where each SECTION
starts, where each paragraph starts and ends. Extract data definition
entries from DATA DIVISION sections. Record COPY statement positions as
stubs (do not expand them).

Produce a single JSON object: the program's raw structural AST.

---

## COBOL fixed-format column rules

Always apply before reading any line:

| Columns (1-based) | Index (0-based) | Content | Action |
|---|---|---|---|
| 1–6 | 0–5 | Sequence number | Ignore |
| 7 | 6 | Indicator | `*` or `/` → skip line (comment); `D` → include |
| 8–11 | 7–10 | Area A | Division/section/01-level headers land here |
| 12–72 | 11–71 | Area B | Statements, paragraph bodies, data clauses |
| 73–80 | 72–79 | Identification | Ignore |

**Area A (columns 8–11):** DIVISION headers, SECTION headers, 01-level and
77-level data entries, and paragraph names all start here.

**Area B (columns 12–72):** Subordinate data entries (02–49 level), PERFORM,
MOVE, IF, EVALUATE, CALL, and all other statements start here.

---

## Phase 1 — Division boundary detection

Scan the entire file line by line. Detect these markers (case-insensitive,
Area A or B):

```bash
# All four divisions
grep -niE "^\s{0,6}[^*/].{0,6}(IDENTIFICATION|ENVIRONMENT|DATA|PROCEDURE)\s+DIVISION" "$FILE"

# Sections within DATA DIVISION
grep -niE "^\s{0,6}[^*/].{0,6}(FILE|WORKING-STORAGE|LOCAL-STORAGE|LINKAGE|COMMUNICATION|REPORT)\s+SECTION" "$FILE"

# PROCEDURE DIVISION (may have USING clause on same or next line)
grep -niE "^\s{0,6}[^*/].{0,6}PROCEDURE\s+DIVISION" "$FILE"
```

Record start line number for each. Compute end line as (next division start - 1)
or end-of-file.

---

## Phase 2 — IDENTIFICATION DIVISION extraction

Within the IDENTIFICATION DIVISION, extract:

```bash
grep -niE "^\s{0,6}[^*/].{0,6}PROGRAM-ID\.\s+([A-Z0-9\-]+)"    "$FILE"
grep -niE "^\s{0,6}[^*/].{0,6}AUTHOR\.\s*(.*)"                  "$FILE"
grep -niE "^\s{0,6}[^*/].{0,6}DATE-WRITTEN\.\s*(.*)"            "$FILE"
grep -niE "^\s{0,6}[^*/].{0,6}DATE-COMPILED\.\s*(.*)"           "$FILE"
```

---

## Phase 3 — DATA DIVISION extraction

### FILE SECTION — FD and SD entries

```bash
# FD (File Description) and SD (Sort Description) headers
grep -niE "^\s{0,6}[^*/].{0,3}(FD|SD)\s+([A-Z0-9\-]+)" "$FILE"
```

For each FD/SD found, record:
- File name (token after FD/SD)
- RECORDING MODE clause if present
- RECORD CONTAINS clause if present
- BLOCK CONTAINS clause if present
- Start line and end line (next FD/SD or next SECTION boundary)

Do NOT parse the 01-level record layout under each FD here — that is
the Data Agent's job. Record only the FD header and its line range.

### WORKING-STORAGE SECTION — data hierarchy

Extract the data item hierarchy using level numbers:

```bash
# All level-numbered entries in WORKING-STORAGE (levels 01–49, 77, 88)
grep -niE "^\s{0,6}[^*/].\s+(0?1|0?2|0?3|0?4|0?5|0?6|0?7|0?8|0?9|[1-4][0-9]|77|88)\s+([A-Z0-9\-]+)" "$FILE"
```

For each entry record:
- Level number (normalised to integer: `01`, `77`, `88`)
- Data name (FILLER if anonymous)
- PIC clause if present on same or continuation line
- OCCURS clause if present (flag as table)
- REDEFINES clause target if present
- VALUE clause if present (literal only — do not evaluate)
- Line number
- Whether it contains a COPY statement instead of inline definition

Level hierarchy rules:
- Level 01 and 77 → top-level entries (children of WORKING-STORAGE)
- Level 02–49 → children of the nearest preceding lower level number
- Level 88 → condition names, children of their parent data item
- FILLER items → record with name `"FILLER"` and a generated unique suffix

### LINKAGE SECTION

Apply the same extraction as WORKING-STORAGE. Mark all entries with
`"section": "LINKAGE"`.

### COPY stubs in DATA DIVISION

Wherever a `COPY` statement appears inside the DATA DIVISION, record a
stub entry instead of expanding it:

```json
{
  "type": "copy_stub",
  "copybook": "CVACT01Y",
  "library": null,
  "line": 145,
  "section": "WORKING-STORAGE",
  "parent_level": "01",
  "note": "Expanded by Data Agent"
}
```

---

## Phase 4 — PROCEDURE DIVISION extraction

### USING clause (parameters)

```bash
# PROCEDURE DIVISION USING parameters
grep -niE "PROCEDURE\s+DIVISION\s+USING\s+(.*)" "$FILE"
```

Record each parameter name listed after USING.

### Section headers within PROCEDURE DIVISION

```bash
# Named sections inside PROCEDURE DIVISION (Area A, followed by SECTION keyword)
grep -niE "^.{7}([A-Z0-9\-]+)\s+SECTION\." "$FILE"
```

### Paragraph names

Paragraph names appear in Area A (columns 8–11 start) and are followed
by a period on the same line or immediately after:

```bash
# Paragraph names — start in Area A, end with period
grep -niE "^.{7}([A-Z0-9\-]+)\s*\." "$FILE"
```

Disambiguate paragraph names from data entries and section headers:
- If the line number falls within DATA DIVISION bounds → skip
- If the token is a known COBOL reserved word → skip
- If the token is immediately followed by `SECTION` → it is a section header

For each paragraph, record:
- Name
- Start line
- End line (line before next paragraph or end of PROCEDURE DIVISION)
- Parent section name (if the paragraph is inside a named section)
- Whether it contains any PERFORM, GO TO, CALL, or STOP RUN statements
  (boolean flags — do not extract the statements themselves here)

### STOP RUN / GOBACK / EXIT PROGRAM detection

```bash
grep -niE "(STOP\s+RUN|GOBACK|EXIT\s+PROGRAM)" "$FILE"
```

Record line numbers — these are terminal points in control flow.

---

## Output schema — single program AST

```json
{
  "program_id": "CBACT01C",
  "source_file": "app/cbl/CBACT01C.CBL",
  "total_lines": 892,
  "identification_division": {
    "start_line": 1,
    "end_line": 18,
    "program_id": "CBACT01C",
    "author": "AWS",
    "date_written": "2022-06-10"
  },
  "environment_division": {
    "start_line": 19,
    "end_line": 45,
    "file_control_entries": [
      { "select_name": "ACCT-FILE", "assign_to": "ACCTFILE", "line": 28 }
    ]
  },
  "data_division": {
    "start_line": 46,
    "end_line": 420,
    "file_section": {
      "start_line": 48,
      "entries": [
        {
          "type": "FD",
          "file_name": "ACCT-FILE",
          "line": 50,
          "end_line": 78,
          "record_contains": "300 CHARACTERS",
          "recording_mode": "F"
        }
      ]
    },
    "working_storage_section": {
      "start_line": 80,
      "entries": [
        {
          "level": 1,
          "name": "WS-RETURN-CODE",
          "pic": "S9(04) COMP",
          "value": "ZEROS",
          "line": 85,
          "redefines": null,
          "occurs": null,
          "children": []
        },
        {
          "level": 1,
          "name": "WS-ACCT-REC",
          "pic": null,
          "line": 90,
          "children": [
            {
              "level": 5,
              "name": "WS-ACCT-ID",
              "pic": "X(11)",
              "line": 92
            }
          ]
        },
        {
          "type": "copy_stub",
          "copybook": "CVACT01Y",
          "library": null,
          "line": 145,
          "section": "WORKING-STORAGE",
          "parent_level": "01"
        }
      ]
    },
    "linkage_section": {
      "start_line": null,
      "entries": []
    }
  },
  "procedure_division": {
    "start_line": 421,
    "end_line": 892,
    "using_parameters": [],
    "sections": [
      {
        "name": "0000-MAIN",
        "start_line": 425,
        "end_line": 530
      }
    ],
    "paragraphs": [
      {
        "name": "0000-MAIN",
        "section": "0000-MAIN",
        "start_line": 425,
        "end_line": 480,
        "has_perform": true,
        "has_goto": false,
        "has_call": false,
        "has_stop_run": false
      },
      {
        "name": "0100-OPEN-FILES",
        "section": "0000-MAIN",
        "start_line": 482,
        "end_line": 510,
        "has_perform": false,
        "has_goto": false,
        "has_call": false,
        "has_stop_run": false
      }
    ],
    "terminal_statements": [
      { "type": "STOP RUN", "line": 888, "paragraph": "9999-EXIT" }
    ]
  },
  "copy_stubs": [
    {
      "copybook": "CVACT01Y",
      "division": "DATA",
      "section": "WORKING-STORAGE",
      "line": 145
    }
  ],
  "parse_issues": []
}
```

---

## Error handling

| Condition | Action |
|---|---|
| File not found | Log `error` in `parse_issues`, return partial result |
| DIVISION boundary not detected | Log `warning`, continue with what was found |
| Ambiguous paragraph name (matches reserved word) | Skip and log `info` |
| PIC clause spans continuation line | Concatenate both lines before extracting |
| Level number out of sequence (e.g. 01 → 05 → 02) | Log `warning`, record as-is |
| Line exceeds column 72 (free-format indicator) | Log `info`, switch to free-format mode for that file |
