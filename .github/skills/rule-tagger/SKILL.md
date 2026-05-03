---
name: rule-tagger
description: >
  Takes classified_conditions.json from the condition-classifier skill and
  promotes each significant condition (signal_strength >= 2) into a named,
  documented business rule. Assigns a stable rule ID, generates a
  human-readable rule name and plain-English description, deduplicates rules
  that appear across multiple programs, groups related rules into rule sets,
  assigns confidence levels, and produces the final rules_artifact.json
  containing the complete business rules catalogue with full source
  traceability. Called by the Rules Agent after condition-classifier.
---

# Skill — rule tagger

## Purpose

Promote each meaningful classified condition into a proper business rule
with a stable identifier, a plain-English name, a description, a
confidence level, and a complete audit trail back to its source. Deduplicate
rules that are logically the same rule implemented in multiple programs.
Group related rules into named rule sets.

Every rule in the output must be traceable to at least one source line.
Every rule name must be understandable by a business analyst without
knowing COBOL.

---

## Phase 1 — filtering

Load `classified_conditions.json`. Apply filters before promotion:

```
Include: signal_strength >= 2
Exclude: signal_strength == 1 (noise) — these go to error_handling_catalogue
         category == UNKNOWN (cannot be named reliably)
```

Noise conditions are not discarded — they are moved to a separate
`error_handling_catalogue` section in the output. The BRD Synthesis
Agent uses them for the error handling chapter.

---

## Phase 2 — rule name generation

For each included condition, generate a rule name using this formula:

```
RULE_NAME = {VERB} + {SUBJECT_FIELD_BUSINESS_NAME} + {QUALIFIER}
```

### VERB selection by category

| Category | Verb |
|---|---|
| `VALIDATION` | `Validate`, `Reject`, `Require`, `Check` |
| `CALCULATION` | `Calculate`, `Compute`, `Apply`, `Derive` |
| `ROUTING` | `Route`, `Select`, `Dispatch` |
| `LIMIT_CHECK` | `Enforce`, `Cap`, `Restrict`, `Allow` |
| `ERROR_HANDLING` | `Handle`, `Detect`, `Catch` |
| `COMPLIANCE` | `Report`, `Flag`, `Enforce compliance for` |

### SUBJECT_FIELD_BUSINESS_NAME

Convert the subject field's COBOL name to a business-readable phrase:
- Strip common WS-, WRK-, IO-, DB-, PROC- prefixes
- Replace hyphens with spaces
- Apply title case
- Expand known abbreviations (see abbreviation table below)

### Abbreviation expansion table

| COBOL abbreviation | Expanded form |
|---|---|
| `ACCT` | Account |
| `TRANS` | Transaction |
| `AMT` | Amount |
| `BAL` | Balance |
| `CUST` | Customer |
| `CARD` | Card |
| `STAT` | Status |
| `TYPE` | Type |
| `CD` or `CODE` | Code |
| `NBR` or `NUM` or `NO` | Number |
| `DT` or `DATE` | Date |
| `ADDR` | Address |
| `PMT` | Payment |
| `AUTH` | Authorisation |
| `EXPIRY` or `EXP` | Expiry |
| `LMT` or `LIMIT` | Limit |
| `CURR` | Currency |
| `INT` | Interest |
| `FEE` | Fee |
| `CR` | Credit |
| `DR` | Debit |
| `CTR` | Counter |
| `MAX` | Maximum |
| `MIN` | Minimum |
| `PCT` or `RATE` | Rate |
| `IND` or `FLAG` or `SW` | Indicator |
| `ERR` | Error |
| `RC` or `RESP` | Response |

### QUALIFIER

Based on the structural pattern:

| Pattern | Qualifier |
|---|---|
| `FIELD_VALUE_COMPARE` | the compared value (e.g. `= 'CR'` → `is Credit`) |
| `RANGE_CHECK` | `within allowed range` or `within [low] to [high]` |
| `NULL_ZERO_CHECK` | `is not zero` / `is not empty` / `is present` |
| `FIELD_FIELD_COMPARE` | `does not exceed [comparand field business name]` |
| `SET_MEMBERSHIP` | `is a valid [subject business name]` |
| `CONDITION_NAME` | use the 88-level name expanded (e.g. `ACCT-OVERDRAWN` → `account is overdrawn`) |
| `ARITHMETIC_RESULT` | `after calculation` |

### Name examples

| COBOL condition | Generated rule name |
|---|---|
| `WS-TRANS-TYPE = 'CR' OR 'DR'` | `Validate transaction type is credit or debit` |
| `ACCT-BALANCE < ZERO` | `Enforce account balance does not go below zero` |
| `WS-TRANS-AMOUNT > WS-CREDIT-LIMIT` | `Enforce transaction amount does not exceed credit limit` |
| `WS-CARD-EXPIRY < WS-CURRENT-DATE` | `Validate card expiry date is not in the past` |
| `EVALUATE WS-TRANS-TYPE` | `Route processing by transaction type` |
| `WS-TRANS-AMOUNT >= 10000` | `Flag transaction amount for compliance reporting` |
| `IF CUST-AGE < 18` | `Validate customer age meets minimum requirement` |

---

## Phase 3 — rule description generation

For each rule, generate a plain-English description (2–4 sentences) using
the pseudocode context, the field types, and the condition outcomes.

Template:
```
{Rule name}. When {subject field} {condition}, the system {THEN outcome}.
{If ELSE exists}: Otherwise, the system {ELSE outcome}.
{Source note}: Implemented in program {PROGRAM-ID}, paragraph {PARA-NAME}.
```

Example:
```
Enforce transaction amount does not exceed credit limit.
When the transaction amount is greater than the customer's credit limit,
the system rejects the transaction and invokes the insufficient-funds
handling routine. Otherwise, the transaction proceeds to balance update.
Implemented in program CBACT01C, paragraph 0300-VALIDATE-AMOUNT.
```

For EVALUATE-based routing rules, describe all branches:
```
Route processing by transaction type.
The system routes each transaction to a dedicated processing path based
on the transaction type code: 'CR' (credit) routes to credit processing,
'DR' (debit) routes to debit processing with balance check enforcement,
and all other type codes are rejected as invalid.
Implemented in program CBACT01C, paragraph 0200-PROCESS-TRANSACTION.
```

---

## Phase 4 — deduplication across programs

The same business rule often appears in multiple programs. Detect and merge:

### Deduplication criteria

Two conditions are the same rule if ALL of the following match:
- Same category
- Same structural pattern
- Same subject field name (or same field normalized_type + length + role)
- Same comparison operator and value (within 10% tolerance for numeric literals)

When a match is found:
- Keep one canonical rule entry
- Add all source locations to a `sources[]` array
- Set `implemented_in_programs` to the full list
- Set `is_duplicated: true`
- Record which program has the most complete implementation as `primary_source`

### Deduplication examples

| Programs | Condition | Action |
|---|---|---|
| CBACT01C and COACT01C both check `ACCT-BALANCE < ZERO` | Same field, same operator, same value | Merge → one rule, two sources |
| CBACT01C checks `AMT > 10000`, COACT01C checks `AMOUNT > 10000` | Same type/length, same operator, same threshold | Merge — different field names, same rule |
| CBACT01C checks `TYPE = 'CR'`, COACT01C checks `TYPE = 'CR' OR 'DR'` | Different value sets | Do NOT merge — record as related rules |

---

## Phase 5 — rule set grouping

Group related rules into named rule sets. A rule set is a logical cluster
of rules that govern the same business domain or process.

### Rule set detection heuristics

Group rules into the same set if:
- They share the same primary subject field (e.g. all rules about `ACCT-BALANCE`)
- They appear in the same paragraph or in closely related paragraphs
- Their names share a common business noun (Account, Transaction, Card, etc.)
- They are all WHEN clauses of the same EVALUATE statement

### Standard rule set names (use these where applicable)

| Rule set name | Contains |
|---|---|
| `Account validation rules` | Rules checking account status, type, existence |
| `Transaction validation rules` | Rules checking transaction type, amount format |
| `Credit limit rules` | Rules enforcing balance and credit limit boundaries |
| `Card management rules` | Rules about card status, expiry, PIN |
| `Authentication rules` | Rules checking user ID, password, session |
| `Transaction routing rules` | Rules selecting processing paths by type/code |
| `Compliance reporting rules` | Rules triggering regulatory reports |
| `Error recovery rules` | Error handling patterns (from error_handling_catalogue) |

If no standard name fits, derive one from the shared subject field name:
`{Subject business name} rules`

---

## Phase 6 — confidence assignment

Each promoted rule gets a final confidence level for the BRD:

| Confidence | Criteria |
|---|---|
| `confirmed` | signal_strength 5, clear field names, obvious THEN/ELSE outcomes |
| `high` | signal_strength 4, business field names, one interpretation |
| `medium` | signal_strength 3, field names partially clear, outcome is clear |
| `low` | signal_strength 2, field names generic or ambiguous, purpose inferred |

All `low` confidence rules are automatically flagged:
`"requires_sme_review": true` with a note explaining what is unclear.

---

## Output schema — rules_artifact.json

```json
{
  "meta": {
    "generated_at": "ISO-8601 timestamp",
    "agent_version": "5_rules@1.0",
    "total_rules": 203,
    "total_rule_sets": 12,
    "requires_sme_review": 18,
    "duplicates_merged": 34
  },
  "stats": {
    "by_category": {
      "VALIDATION": 89,
      "CALCULATION": 44,
      "ROUTING": 38,
      "LIMIT_CHECK": 21,
      "COMPLIANCE": 6
    },
    "by_confidence": {
      "confirmed": 98,
      "high": 57,
      "medium": 30,
      "low": 18
    }
  },
  "rule_sets": [
    {
      "rule_set_id": "RS-001",
      "name": "Credit limit rules",
      "description": "Rules that enforce transaction and balance limits relative to customer credit limits.",
      "rule_count": 5,
      "rule_ids": ["BR-001", "BR-002", "BR-003", "BR-004", "BR-005"],
      "primary_programs": ["CBACT01C", "COACT02C"]
    }
  ],
  "business_rules": [
    {
      "rule_id": "BR-001",
      "rule_set_id": "RS-001",
      "name": "Enforce transaction amount does not exceed credit limit",
      "category": "LIMIT_CHECK",
      "confidence": "confirmed",
      "requires_sme_review": false,
      "description": "Enforce transaction amount does not exceed credit limit. When the transaction amount is greater than the customer credit limit, the system rejects the transaction and routes to the insufficient-funds handling routine. Otherwise, the transaction proceeds to balance update processing. Implemented in programs CBACT01C and COACT02C.",
      "condition": {
        "text": "WS-TRANS-AMOUNT > WS-CREDIT-LIMIT",
        "pattern": "FIELD_FIELD_COMPARE",
        "subject_field": "WS-TRANS-AMOUNT",
        "threshold_field": "WS-CREDIT-LIMIT"
      },
      "outcomes": {
        "when_true": "Route to 8000-INSUFFICIENT-FUNDS",
        "when_false": "Proceed to balance update"
      },
      "is_duplicated": true,
      "primary_source": {
        "program_id": "CBACT01C",
        "paragraph": "0300-VALIDATE-AMOUNT",
        "line": 680
      },
      "sources": [
        {
          "program_id": "CBACT01C",
          "paragraph": "0300-VALIDATE-AMOUNT",
          "line": 680,
          "condition_id": "COND-CBACT01C-0300-001"
        },
        {
          "program_id": "COACT02C",
          "paragraph": "0250-CHECK-LIMITS",
          "line": 412,
          "condition_id": "COND-COACT02C-0250-001"
        }
      ],
      "implemented_in_programs": ["CBACT01C", "COACT02C"],
      "related_rules": ["BR-002", "BR-003"]
    },
    {
      "rule_id": "BR-002",
      "rule_set_id": "RS-001",
      "name": "Enforce account balance does not go below zero",
      "category": "LIMIT_CHECK",
      "confidence": "confirmed",
      "requires_sme_review": false,
      "description": "Enforce account balance does not go below zero. When a debit transaction would cause the account balance to become negative, the system halts the debit and applies the overdraft handling procedure. This rule is equivalent to the ACCT-OVERDRAWN condition defined in copybook CVACT01Y.",
      "condition": {
        "text": "ACCT-BALANCE < ZERO",
        "pattern": "NULL_ZERO_CHECK",
        "subject_field": "ACCT-BALANCE",
        "linked_88_condition": "ACCT-OVERDRAWN"
      },
      "outcomes": {
        "when_true": "Route to overdraft handling",
        "when_false": "Proceed with debit"
      },
      "is_duplicated": false,
      "primary_source": {
        "program_id": "CBACT01C",
        "paragraph": "0200-PROCESS-TRANSACTION",
        "line": 558
      },
      "sources": [
        {
          "program_id": "CBACT01C",
          "paragraph": "0200-PROCESS-TRANSACTION",
          "line": 558,
          "condition_id": "COND-CBACT01C-0200-002"
        }
      ],
      "implemented_in_programs": ["CBACT01C"],
      "related_rules": ["BR-001"]
    },
    {
      "rule_id": "BR-047",
      "rule_set_id": "RS-005",
      "name": "Route processing by transaction type",
      "category": "ROUTING",
      "confidence": "high",
      "requires_sme_review": false,
      "description": "Route processing by transaction type. The system routes each incoming transaction to a dedicated processing path determined by the two-character transaction type code. Credit transactions ('CR') are routed to credit processing. Debit transactions ('DR') are routed to debit processing with credit limit enforcement. All other codes are rejected as invalid transaction types.",
      "condition": {
        "text": "EVALUATE WS-TRANS-TYPE",
        "pattern": "SET_MEMBERSHIP",
        "subject_field": "WS-TRANS-TYPE",
        "when_clauses": [
          { "value": "'CR'", "outcome": "Route to 1000-PROCESS-CREDIT" },
          { "value": "'DR'", "outcome": "Route to 1100-PROCESS-DEBIT" },
          { "value": "OTHER", "outcome": "Route to 9900-INVALID-TRANS-TYPE" }
        ]
      },
      "is_duplicated": false,
      "primary_source": {
        "program_id": "CBACT01C",
        "paragraph": "0200-PROCESS-TRANSACTION",
        "line": 520
      },
      "sources": [
        {
          "program_id": "CBACT01C",
          "paragraph": "0200-PROCESS-TRANSACTION",
          "line": 520,
          "condition_id": "COND-CBACT01C-0200-001"
        }
      ],
      "implemented_in_programs": ["CBACT01C"]
    },
    {
      "rule_id": "BR-098",
      "rule_set_id": "RS-009",
      "name": "Calculate interest on outstanding balance",
      "category": "CALCULATION",
      "confidence": "medium",
      "requires_sme_review": true,
      "sme_review_note": "Interest rate source is unclear — WS-INT-RATE may be loaded from a parameter file or table not visible in this program. SME should confirm rate source and compounding method.",
      "description": "Calculate interest on outstanding balance. The system computes interest by multiplying the current account balance by the interest rate and dividing by 100. The result is rounded and added to the balance. The exact rate source and compounding frequency require SME confirmation.",
      "condition": {
        "text": "COMPUTE WS-INTEREST ROUNDED = ACCT-BALANCE * WS-INT-RATE / 100",
        "pattern": "ARITHMETIC_RESULT",
        "subject_field": "WS-INTEREST",
        "operand_fields": ["ACCT-BALANCE", "WS-INT-RATE"]
      },
      "is_duplicated": false,
      "primary_source": {
        "program_id": "CBINT01C",
        "paragraph": "1000-CALC-INTEREST",
        "line": 342
      },
      "sources": [
        {
          "program_id": "CBINT01C",
          "paragraph": "1000-CALC-INTEREST",
          "line": 342,
          "condition_id": "COND-CBINT01C-1000-001"
        }
      ],
      "implemented_in_programs": ["CBINT01C"]
    }
  ],
  "error_handling_catalogue": [
    {
      "error_id": "EH-001",
      "type": "FILE_IO_ERROR",
      "condition_text": "WS-FILE-STATUS NOT = '00'",
      "program_id": "CBACT01C",
      "paragraph": "0200-PROCESS-TRANSACTION",
      "line": 612,
      "handler_paragraph": "9900-FILE-ERROR",
      "description": "Standard file I/O status check after ACCT-FILE read operation."
    }
  ],
  "rules_by_program": {
    "CBACT01C": ["BR-001", "BR-002", "BR-047"],
    "COACT02C": ["BR-001"],
    "CBINT01C": ["BR-098"]
  },
  "issues": [
    {
      "severity": "info",
      "type": "sme_review_required",
      "rule_id": "BR-098",
      "message": "Interest rate source unresolvable statically — SME confirmation needed"
    }
  ]
}
```

---

## Error handling

| Condition | Action |
|---|---|
| No rule name can be generated (all field names generic) | Use `Rule from {PROGRAM-ID} line {N}` as fallback name, set confidence `low` |
| Deduplication candidate fields have different lengths | Do not merge — record as `related_rule` link only |
| Rule set cannot be determined | Assign to catch-all set `Uncategorised rules` |
| Outcome text is missing from branch | Record outcomes as `unknown`, flag `requires_sme_review: true` |
| More than 20 WHEN clauses in one EVALUATE | Summarise top 5, append `... and {N} more` note |
| Duplicate merge produces conflicting descriptions | Keep both descriptions, flag discrepancy for SME |
