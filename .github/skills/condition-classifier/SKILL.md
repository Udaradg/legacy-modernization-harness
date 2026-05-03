---
name: condition-classifier
description: >
  Reads every branch entry from the logic_artifact branch_map and every
  88-level condition from the data_artifact condition_catalogue. Examines
  each condition's fields, operators, values, and context to classify it
  into a rule category (validation, calculation, routing, limit-check,
  error-handling, compliance) and a structural pattern (field-value-compare,
  range-check, null-check, status-check, arithmetic-result, multi-condition).
  Produces classified_conditions.json as intermediate output for the
  rule-tagger skill. Called by the Rules Agent before rule-tagger.
tools: Read, Bash
---

# Skill — condition classifier

## Purpose

Examine every conditional branch extracted by the Logic Agent and every
88-level condition from the Data Agent. For each one, determine:
1. What rule category it belongs to
2. What structural pattern it uses
3. Which fields are involved and what role they play
4. How strong a signal it is for a genuine business rule

Produce a classified list ready for the rule-tagger to promote into
named, documented rules.

Do NOT name or document rules yet — classify structure and pattern only.

---

## Input sources

### Source 1 — logic_artifact branch_map

For each program in `logic_artifact.programs`, load its
`program_logic/{PROGRAM-ID}_logic.json` and read the `branch_map` array.

Each branch entry contains:
- `type`: IF / EVALUATE / AT_END / INVALID_KEY / ON_EXCEPTION
- `condition_text`: raw condition string from COBOL source
- `condition_fields`: list of field names referenced
- `outcomes`: THEN/ELSE or WHEN clause bodies
- `paragraph`: which paragraph it lives in
- `line`: source line number

### Source 2 — data_artifact condition_catalogue

Each 88-level condition entry contains:
- `condition_name`: the COBOL 88-level name
- `parent_field`: the field it tests
- `values`: the value set (literals, ranges)
- `programs_in_scope`: programs that can reference it

### Source 3 — data_artifact field_catalogue

Used to look up field metadata for each field in `condition_fields`:
- `normalized_type`: STRING / INTEGER / DECIMAL / etc.
- `length`: field length
- `is_condition`: whether it is itself a condition name
- Field name patterns (suffix analysis)

---

## Phase 1 — rule category classification

For each condition, assign one primary category. Use the decision tree below.

### Category 1 — VALIDATION

A condition that checks whether input data is acceptable before processing.

Signals:
- Condition field name contains: `TYPE`, `CODE`, `STATUS`, `FLAG`,
  `SWITCH`, `IND`, `INDICATOR`, `CLASS`, `KIND`
- Condition checks a field against a fixed set of valid values
  (especially using EVALUATE or a group of 88-level conditions)
- Condition is in a paragraph whose name contains: `VALIDATE`, `EDIT`,
  `CHECK`, `VERIFY`, `SCRN`, `SCREEN`
- Condition is evaluated before any WRITE or UPDATE operation
- 88-level condition on a type/status field

Pattern examples:
```
IF TRANS-TYPE NOT IN ('CR', 'DR', 'RF')  → VALIDATION
IF ACCT-STATUS = 'CLOSED'                → VALIDATION
EVALUATE WS-INPUT-CODE WHEN ...          → VALIDATION (if subject is a code field)
IF CUST-AGE < 0 OR CUST-AGE > 150       → VALIDATION (range plausibility check)
```

### Category 2 — CALCULATION

A condition that guards or results from an arithmetic operation.

Signals:
- Condition fields are numeric (`INTEGER`, `DECIMAL`, `SIGNED_DECIMAL`)
- Condition follows a COMPUTE, ADD, SUBTRACT, MULTIPLY, or DIVIDE
- Condition uses arithmetic comparison: `>`, `<`, `>=`, `<=`, `=` on numeric field
- ON SIZE ERROR clause after arithmetic
- Paragraph name contains: `CALC`, `COMPUTE`, `TOTAL`, `ACCUM`, `RATE`,
  `INTEREST`, `TAX`, `FEE`, `PREMIUM`, `CHARGE`, `AMOUNT`

Pattern examples:
```
IF WS-TOTAL > WS-LIMIT             → CALCULATION (result check)
ON SIZE ERROR PERFORM 9900-OVERFLOW → CALCULATION (arithmetic guard)
IF WS-RATE = ZEROS                  → CALCULATION (zero-divisor guard)
COMPUTE ROUNDED                     → CALCULATION (rounding rule)
```

### Category 3 — ROUTING

A condition that determines which process path or program to invoke next.

Signals:
- Condition directly controls a PERFORM target selection
- EVALUATE subject is a transaction type, function code, or menu selection
- Condition field name contains: `TRANS-TYPE`, `FUNC-CODE`, `MENU`,
  `OPTION`, `REQUEST`, `ACTION`, `COMMAND`, `EIBAID` (CICS attention ID)
- Condition appears in a paragraph named: `ROUTE`, `DISPATCH`, `SWITCH`,
  `SELECT`, `MAIN`, `DRIVER`, `CONTROL`
- GO TO DEPENDING ON (always routing)

Pattern examples:
```
EVALUATE WS-TRANS-TYPE WHEN 'CR' ... WHEN 'DR'  → ROUTING
GO TO P1 P2 P3 DEPENDING ON WS-OPTION           → ROUTING
IF EIBAID = DFHPF1                               → ROUTING (CICS key routing)
EVALUATE EIBCALEN WHEN ZERO ...                  → ROUTING (CICS first-time flag)
```

### Category 4 — LIMIT CHECK

A condition that enforces a business threshold, ceiling, or floor.

Signals:
- Condition compares a value to a threshold that appears as a literal,
  a constant field (VALUE clause), or a parameter field
- Field name contains: `LIMIT`, `MAX`, `MIN`, `THRESH`, `CAP`,
  `CEILING`, `FLOOR`, `CREDIT-LIMIT`, `OVERDRAFT`, `BALANCE`
- Condition uses `>`, `<`, `>=`, `<=` on an amount or count field
- The compared literal is a non-trivial value (not ZERO, SPACES, 1)

Pattern examples:
```
IF WS-AMOUNT > CREDIT-LIMIT              → LIMIT_CHECK
IF ACCT-BALANCE < ZERO                   → LIMIT_CHECK (overdraft)
IF WS-RETRY-COUNT >= 3                   → LIMIT_CHECK (retry cap)
IF WS-TRANS-AMOUNT > 10000               → LIMIT_CHECK (reporting threshold)
```

### Category 5 — ERROR HANDLING

A condition that detects and responds to system, I/O, or program errors.

Signals:
- AT END, INVALID KEY, ON EXCEPTION, ON OVERFLOW clauses
- Condition field name contains: `STATUS`, `RESP`, `SQLCODE`, `RETURN-CODE`,
  `RC`, `ERR`, `ERROR`, `ABEND`, `EXCEPTION`
- Condition tests a file status code (2-character field, PIC XX)
- Condition is in a paragraph named: `ERROR`, `ABEND`, `EXCEPTION`,
  `HANDLER`, `ROUTINE`, `9999`, `9900`, `ERR`
- Condition tests CICS RESP field against DFHRESP values
- Condition tests SQLCODE against 0, 100, or negative values

Pattern examples:
```
AT END MOVE 'Y' TO WS-EOF                       → ERROR_HANDLING (I/O)
IF WS-FILE-STATUS NOT = '00'                    → ERROR_HANDLING
IF SQLCODE NOT = 0                              → ERROR_HANDLING
IF WS-CICS-RESP NOT = DFHRESP(NORMAL)           → ERROR_HANDLING
ON SIZE ERROR PERFORM 9900-OVERFLOW             → ERROR_HANDLING
```

### Category 6 — COMPLIANCE

A condition that enforces a regulatory, security, or audit requirement.

Signals (these are heuristic — flag for SME confirmation):
- Field name contains: `AUDIT`, `LOG`, `TRACE`, `SECURITY`, `AUTH`,
  `PERMISSION`, `RESTRICT`, `REGULATORY`, `REPORT`, `CURRENCY`
- Condition checks amounts against regulatory thresholds (e.g. 10000
  for CTR/SAR reporting in banking)
- Condition involves date comparison against a regulatory date
- Paragraph name contains: `AUDIT`, `LOG`, `SECURE`, `COMPLY`, `REPORT`
- Program is in a batch job with regulatory-sounding name

Pattern examples:
```
IF WS-TRANS-AMOUNT >= 10000            → COMPLIANCE (CTR threshold — flag)
IF WS-USER-ID NOT = WS-AUTHORISED-ID  → COMPLIANCE (access control)
IF TRANS-DATE < EFFECTIVE-DATE         → COMPLIANCE (date rule — flag)
```

---

## Phase 2 — structural pattern classification

For each condition, assign a structural pattern — independent of category.

| Pattern | Description | Detection |
|---|---|---|
| `FIELD_VALUE_COMPARE` | Field compared to a literal value | `field = 'X'` or `field = 123` |
| `FIELD_FIELD_COMPARE` | Two fields compared to each other | `field-a = field-b` |
| `RANGE_CHECK` | Field compared to two bounds | `field >= low AND field <= high` or THRU in 88-level |
| `NULL_ZERO_CHECK` | Field tested for empty/zero/spaces | `field = ZERO`, `field = SPACES`, `field = LOW-VALUES` |
| `STATUS_CODE_CHECK` | Field tested against a status/return code | file status, SQLCODE, CICS RESP |
| `SET_MEMBERSHIP` | Field tested against a list of values | `EVALUATE` WHEN list, or `field IN (a b c)` |
| `ARITHMETIC_RESULT` | Condition based on result of calculation | follows COMPUTE/ADD/SUBTRACT |
| `MULTI_CONDITION` | Compound AND/OR condition | two or more sub-conditions joined |
| `CONDITION_NAME` | 88-level condition name used directly | `IF ACCT-OVERDRAWN` |
| `NEGATION` | NOT prefix on any other pattern | `IF NOT VALID-TRANS-TYPE` |

---

## Phase 3 — field role analysis

For each field in `condition_fields`, determine its role in the condition:

| Role | Description | How to detect |
|---|---|---|
| `SUBJECT` | The field being tested | Left side of comparison, or EVALUATE subject |
| `COMPARAND` | Another field being compared to subject | Right side of field-field compare |
| `THRESHOLD` | A fixed limit or constant | Right side of range/limit check with VALUE clause |
| `STATUS_FIELD` | A system or I/O status code | Name contains STATUS, RESP, SQLCODE, RC |
| `FLAG_FIELD` | A boolean indicator field | PIC X(1) or 88-level condition, name ends in FLAG/IND/SW |
| `ACCUMULATOR` | A running total or counter | Name contains COUNT, TOTAL, ACCUM, SUM, CTR |
| `KEY_FIELD` | A record identifier field | Name ends in ID, KEY, NBR, NUM, NO |

---

## Phase 4 — signal strength scoring

Score each classified condition from 1–5 for how likely it represents a
genuine, named business rule (vs. a technical/plumbing condition).

| Score | Label | Criteria |
|---|---|---|
| 5 | `certain` | 88-level condition name used directly; or EVALUATE on a business type/code field |
| 4 | `high` | Named threshold comparison on a business amount field; clear business field names |
| 3 | `medium` | Condition on typed field with business-sounding name; category is clear |
| 2 | `low` | Generic field names; category inferred; purpose unclear |
| 1 | `noise` | File status check, CICS RESP check, SQLCODE check — technical plumbing, not business rules |

Score 1 (`noise`) conditions are still recorded but excluded from
rule-tagger processing by default (they appear in the error-handling
section of the BRD, not the business rules section).

---

## Output schema — classified_conditions.json

```json
{
  "meta": {
    "generated_at": "ISO-8601 timestamp",
    "agent_version": "5_rules@1.0",
    "total_branches_examined": 892,
    "total_conditions_classified": 748,
    "noise_excluded": 144
  },
  "classified_conditions": [
    {
      "condition_id": "COND-CBACT01C-0200-001",
      "program_id": "CBACT01C",
      "paragraph": "0200-PROCESS-TRANSACTION",
      "source_line": 545,
      "branch_type": "IF",
      "condition_text": "WS-TRANS-TYPE = 'CR'",
      "pseudocode_context": "SELECT CASE transaction_type WHEN 'CR'",
      "category": "ROUTING",
      "structural_pattern": "FIELD_VALUE_COMPARE",
      "fields": [
        {
          "name": "WS-TRANS-TYPE",
          "role": "SUBJECT",
          "normalized_type": "STRING",
          "length": 2
        }
      ],
      "signal_strength": 4,
      "signal_label": "high",
      "notes": "Controls routing to credit vs debit processing path"
    },
    {
      "condition_id": "COND-CBACT01C-0200-002",
      "program_id": "CBACT01C",
      "paragraph": "0200-PROCESS-TRANSACTION",
      "source_line": 558,
      "branch_type": "IF",
      "condition_text": "ACCT-BALANCE < ZERO",
      "pseudocode_context": "IF account_balance < 0 THEN",
      "category": "LIMIT_CHECK",
      "structural_pattern": "NULL_ZERO_CHECK",
      "fields": [
        {
          "name": "ACCT-BALANCE",
          "role": "SUBJECT",
          "normalized_type": "SIGNED_DECIMAL",
          "length": 12,
          "decimal_places": 2
        }
      ],
      "signal_strength": 5,
      "signal_label": "certain",
      "notes": "Overdraft detection — 88-level ACCT-OVERDRAWN maps to this condition"
    },
    {
      "condition_id": "COND-CBACT01C-0200-003",
      "program_id": "CBACT01C",
      "paragraph": "0200-PROCESS-TRANSACTION",
      "source_line": 612,
      "branch_type": "IF",
      "condition_text": "WS-FILE-STATUS NOT = '00'",
      "pseudocode_context": "IF file_status != '00'",
      "category": "ERROR_HANDLING",
      "structural_pattern": "STATUS_CODE_CHECK",
      "fields": [
        {
          "name": "WS-FILE-STATUS",
          "role": "STATUS_FIELD",
          "normalized_type": "STRING",
          "length": 2
        }
      ],
      "signal_strength": 1,
      "signal_label": "noise",
      "notes": "Standard file I/O status check — technical plumbing"
    },
    {
      "condition_id": "COND-CBACT01C-0300-001",
      "program_id": "CBACT01C",
      "paragraph": "0300-VALIDATE-AMOUNT",
      "source_line": 680,
      "branch_type": "IF",
      "condition_text": "WS-TRANS-AMOUNT > WS-CREDIT-LIMIT",
      "pseudocode_context": "IF transaction_amount > credit_limit THEN",
      "category": "LIMIT_CHECK",
      "structural_pattern": "FIELD_FIELD_COMPARE",
      "fields": [
        {
          "name": "WS-TRANS-AMOUNT",
          "role": "SUBJECT",
          "normalized_type": "SIGNED_DECIMAL",
          "length": 10,
          "decimal_places": 2
        },
        {
          "name": "WS-CREDIT-LIMIT",
          "role": "THRESHOLD",
          "normalized_type": "SIGNED_DECIMAL",
          "length": 10,
          "decimal_places": 2
        }
      ],
      "signal_strength": 5,
      "signal_label": "certain",
      "notes": "Credit limit enforcement — field names are explicit"
    }
  ],
  "by_category": {
    "VALIDATION": 89,
    "CALCULATION": 44,
    "ROUTING": 38,
    "LIMIT_CHECK": 21,
    "ERROR_HANDLING": 11,
    "COMPLIANCE": 6,
    "noise": 144
  },
  "classifier_issues": []
}
```

---

## Error handling

| Condition | Action |
|---|---|
| Branch has no `condition_fields` | Classify from `condition_text` alone, set signal_strength 2, log `warning` |
| Field not found in `field_catalogue` | Use raw field name, omit type metadata, log `info` |
| Category is ambiguous between two options | Assign primary, record secondary in `notes`, log `info` |
| Condition text is empty or unparseable | Set category `UNKNOWN`, signal_strength 1, log `warning` |
| 88-level condition name with no parent field | Flag as `orphan_condition`, log `warning` |
