---
name: data-dictionary-builder
description: >
  Takes all layout JSON files produced by the record-layout-parser skill and
  normalises them into a unified, codebase-wide data dictionary. Deduplicates
  shared copybook structures, resolves REDEFINES alias chains, maps fields to
  the programs that read or write them, identifies candidate entity
  relationships between records (shared key fields, foreign key patterns),
  and produces the data_artifact.json containing the full data dictionary,
  a flat field catalogue, and an ERD-ready entity-relationship model.
  Called by the Data Agent after record-layout-parser.
---

# Skill — data dictionary builder

## Purpose

Aggregate all per-layout JSON files into a single, unified data model.
Normalise duplicates, resolve REDEFINES chains, enrich each field with
cross-program usage metadata, detect candidate relationships between records,
and produce the final data artifact that downstream agents use as their
authoritative data reference.

Do NOT interpret field purpose or business meaning — use field names,
types, and structural patterns only to infer relationships.

---

## Phase 1 — record deduplication

Copybooks are parsed once but used by many programs. Build a master record
catalogue where each copybook layout appears exactly once.

### Deduplication rules

```
1. Load all layout JSON files from data_layouts/
2. Separate into two groups:
   a. Copybook layouts  (source_type == "copybook")
   b. Program WS layouts (source_type == "program_ws")
3. For copybook layouts: use the copybook ID as the canonical record ID
   (e.g. CVACT01Y → one entry in master catalogue)
4. For program WS layouts: each 01-level group becomes its own record entry
   - If an 01-level group is just a COPY stub expansion, point to the
     canonical copybook record rather than duplicating
5. Assign each record a stable unique ID:
   format: REC-{SOURCE_ID}-{RECORD_NAME}
   e.g.:   REC-CVACT01Y-ACCOUNT-RECORD
           REC-CBACT01C_WS-WS-CONTROL-BLOCK
```

---

## Phase 2 — flat field catalogue

Flatten every record's nested hierarchy into a flat list of elementary
fields (leaf nodes only — group items are retained as structural metadata
but not as individual catalogue entries).

For each elementary field produce a catalogue entry:

```
field_id         : globally unique ID
                   format: FLD-{RECORD_ID}-{FIELD_NAME}
                   e.g.:   FLD-REC-CVACT01Y-ACCOUNT-RECORD-ACCT-ID
record_id        : parent record ID
field_name       : COBOL name (uppercase)
qualified_name   : full dotted path from 01-level
                   e.g.: ACCOUNT-RECORD.ACCT-ID
level            : level number (integer)
normalized_type  : from PIC parsed output
length           : total character/byte length
decimal_places   : decimal digits (0 if integer or string)
signed           : boolean
storage_type     : DISPLAY / BINARY / PACKED_DECIMAL / etc.
is_filler        : boolean
is_condition     : boolean (true for 88-level entries)
redefines        : field_id of redefined field, or null
occurs_max       : max occurs, or null if not a table entry
source_line      : original line in source file
```

### Handling REDEFINES in the flat catalogue

When field B REDEFINES field A:
- Both A and B appear in the flat catalogue as separate entries
- B's entry has `redefines: "FLD-...-A"`
- Add a `redefines_group` object listing all fields that share the same
  storage (A and all its redefines siblings)

```json
"redefines_groups": [
  {
    "storage_anchor": "FLD-REC-CVACT01Y-ACCOUNT-RECORD-ACCT-TYPE-CODE",
    "aliases": [
      "FLD-REC-CVACT01Y-ACCOUNT-RECORD-ACCT-TYPE-REDEF"
    ],
    "bytes": 3,
    "note": "Same 3 bytes interpreted as X(3) or 9(3)"
  }
]
```

---

## Phase 3 — cross-program field usage mapping

For each field in the flat catalogue, determine which programs use it
and in what role.

### Usage detection strategy

Using the inventory artifact's `copybook_map`:
- If a copybook is used by programs A, B, C → all fields in that copybook
  are potentially used by A, B, C
- Record usage as `"potential"` — exact read/write analysis is the
  Logic Agent's job

Mark fields from widely-shared copybooks (used by 5+ programs) as
`"high_shared"` — these are the most important fields for the BRD
data model chapter.

```json
"field_usage": {
  "field_id": "FLD-REC-CVACT01Y-ACCOUNT-RECORD-ACCT-ID",
  "used_by_programs": ["CBACT01C", "COACT01C", "COACT02C"],
  "usage_type": "potential",
  "shared_level": "high_shared"
}
```

---

## Phase 4 — entity relationship detection

Identify candidate relationships between records using structural heuristics.
These are candidates only — the Logic Agent confirms them.

### Heuristic 1 — shared key field names

If two different records both contain a field with the same name and the
same PIC clause (type + length), they likely share a key relationship.

```
For each pair of records (A, B):
  For each field in A:
    If B contains a field with:
      - Same name (exact match, case-insensitive)
      - Same normalized_type
      - Same length
    → candidate relationship: A relates to B via field-name
```

### Heuristic 2 — naming convention patterns

Common COBOL key field naming patterns signal relationships:

| Pattern | Likely meaning |
|---|---|
| Field ends with `-ID` | Primary or foreign key identifier |
| Field ends with `-KEY` | Key field |
| Field ends with `-NBR` or `-NUM` or `-NO` | Numeric identifier |
| Field ends with `-CODE` or `-CD` | Code/lookup key |
| Field ends with `-DATE` or `-DT` | Date field — link temporal records |
| Field name contains another record's base name | Foreign key candidate |

For each field matching a key pattern, check if any other record has a
field with the same name → candidate relationship.

### Heuristic 3 — FD to WS record pairing

Each FD entry in the FILE SECTION declares a file record. The corresponding
WORKING-STORAGE section often has a matching 01-level group that mirrors
the FD record layout (used as an I/O buffer).

Detect these pairs:
- FD `ACCT-FILE` → look for WS group `WS-ACCT-REC` or `ACCT-RECORD` or
  similar (strip common prefixes WS-, WRK-, IO-, BUF-)
- If found → record as a `FILE_BUFFER` relationship

### Relationship types

| Type | Meaning |
|---|---|
| `SHARED_KEY` | Two records share a common key field name and type |
| `FILE_BUFFER` | WS record is the I/O buffer for an FD file record |
| `REDEFINES_ALIAS` | Same storage interpreted differently |
| `COPY_EXPANSION` | Record is an expansion of a copybook |
| `NAMING_CONVENTION` | Key naming pattern suggests relationship |

---

## Phase 5 — 88-level condition catalogue

Build a separate catalogue of all 88-level condition names across the
codebase. These represent coded values and are critical for the Rules Agent.

For each 88-level entry record:
- Condition name
- Parent field name and record
- Value set (values or ranges)
- All programs that have access to this condition (via copybook usage)

Group conditions by parent field — a field with many 88-level children
is a status/type/code field and is high priority for business rule extraction.

```json
"condition_catalogue": [
  {
    "condition_name": "ACCT-OVERDRAWN",
    "parent_field": "ACCT-BALANCE",
    "parent_record": "REC-CVACT01Y-ACCOUNT-RECORD",
    "values": [{"from": -999999999.99, "to": -0.01}],
    "programs_in_scope": ["CBACT01C", "COACT01C"],
    "condition_group_size": 4,
    "note": "4 conditions on ACCT-BALANCE — likely a status field"
  }
]
```

---

## Output schema — data_artifact.json

```json
{
  "meta": {
    "generated_at": "ISO-8601 timestamp",
    "agent_version": "3_data@1.0",
    "source_artifacts": ["inventory_artifact.json", "parser_artifact.json"],
    "total_records": 187,
    "total_fields": 2341,
    "total_conditions": 312
  },
  "stats": {
    "copybook_records": 31,
    "program_ws_records": 156,
    "redefines_groups": 48,
    "occurs_tables": 23,
    "high_shared_fields": 89,
    "candidate_relationships": 34,
    "unresolved_copy_stubs": 3
  },
  "record_catalogue": [
    {
      "record_id": "REC-CVACT01Y-ACCOUNT-RECORD",
      "record_name": "ACCOUNT-RECORD",
      "source_id": "CVACT01Y",
      "source_type": "copybook",
      "source_file": "app/copy/CVACT01Y.CPY",
      "used_by_programs": ["CBACT01C", "COACT01C", "COACT02C"],
      "total_fields": 24,
      "total_bytes": 300,
      "has_redefines": true,
      "has_occurs": false,
      "field_ids": ["FLD-REC-CVACT01Y-ACCOUNT-RECORD-ACCT-ID"]
    }
  ],
  "field_catalogue": [
    {
      "field_id": "FLD-REC-CVACT01Y-ACCOUNT-RECORD-ACCT-ID",
      "record_id": "REC-CVACT01Y-ACCOUNT-RECORD",
      "field_name": "ACCT-ID",
      "qualified_name": "ACCOUNT-RECORD.ACCT-ID",
      "level": 5,
      "normalized_type": "STRING",
      "length": 11,
      "decimal_places": 0,
      "signed": false,
      "storage_type": "DISPLAY",
      "pic_raw": "X(11)",
      "is_filler": false,
      "is_condition": false,
      "redefines": null,
      "occurs_max": null,
      "source_line": 7,
      "used_by_programs": ["CBACT01C", "COACT01C", "COACT02C"],
      "shared_level": "high_shared"
    }
  ],
  "redefines_groups": [],
  "condition_catalogue": [],
  "data_model": {
    "entities": [
      {
        "entity_id": "ENT-ACCOUNT-RECORD",
        "display_name": "Account record",
        "primary_record": "REC-CVACT01Y-ACCOUNT-RECORD",
        "key_fields": ["ACCT-ID"],
        "field_count": 24,
        "programs_using": ["CBACT01C", "COACT01C", "COACT02C"]
      }
    ],
    "relationships": [
      {
        "relationship_id": "REL-001",
        "type": "SHARED_KEY",
        "from_entity": "ENT-ACCOUNT-RECORD",
        "to_entity": "ENT-CARD-RECORD",
        "via_field": "ACCT-ID",
        "confidence": "high",
        "note": "Both records contain ACCT-ID X(11)"
      },
      {
        "relationship_id": "REL-002",
        "type": "FILE_BUFFER",
        "from_entity": "ENT-ACCT-FILE-FD",
        "to_entity": "ENT-WS-ACCT-BUFFER",
        "confidence": "high",
        "note": "FD ACCT-FILE ↔ WS-ACCT-REC pattern match"
      }
    ]
  },
  "issues": [
    {
      "severity": "warning",
      "type": "unresolved_copy_stub",
      "program": "CBACT01C",
      "copybook": "EXTCOPY1",
      "message": "Copybook EXTCOPY1 not found in registry — stub left unresolved"
    }
  ]
}
```

---

## Error handling

| Condition | Action |
|---|---|
| Layout file for a program not found | Log `error`, skip that program's WS, continue |
| Two records have identical names (different sources) | Append source ID suffix to disambiguate, log `warning` |
| REDEFINES chain is circular | Log `error`, break chain at the cycle point |
| Field qualified name collision after REPLACING | Append `_ALT` suffix, log `warning` |
| Relationship candidate has low field count match | Set `confidence: "low"`, include but flag |
| Condition values cannot be parsed (complex expression) | Record raw text, set `parsed: false`, log `info` |
