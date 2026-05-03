---
name: pseudocode-generator
description: >
  Takes each paragraph's raw COBOL source lines and the annotated execution
  trace from the control-flow-tracer skill, then translates every COBOL
  statement into structured, readable pseudocode. Resolves field references
  using the data_artifact field catalogue, explains COBOL-specific idioms
  inline, annotates I/O operations with file and record context, and flags
  unresolvable dynamic constructs for SME review. Produces a pseudocode
  block per paragraph and a program-level narrative summary. Called by the
  Logic Agent after control-flow-tracer.
---

# Skill — pseudocode generator

## Purpose

For each paragraph in a program, read the raw COBOL source lines and
translate every statement into plain, structured pseudocode. Use the
field catalogue from the Data Agent to annotate field references with
their type and context. Use the execution trace from the control-flow-tracer
to understand the branching structure before translating.

The output pseudocode must be readable by a business analyst who does not
know COBOL. Explain idioms in inline annotations. Never leave raw COBOL
in the pseudocode output without translation.

---

## Pseudocode style conventions

- Use `IF / ELSE IF / ELSE / END IF` for conditional blocks
- Use `WHILE condition DO / END WHILE` for PERFORM UNTIL (test before)
- Use `DO / WHILE condition END DO` for PERFORM WITH TEST AFTER
- Use `FOR index FROM x TO y STEP z / END FOR` for PERFORM VARYING
- Use `CALL program-name (params)` for static CALL statements
- Use `CALL [dynamic: variable-name] (params)` for dynamic CALL statements
- Use `READ file INTO record` for file READ operations
- Use `WRITE record TO file` for file WRITE operations
- Use `MOVE value TO field` only for significant data movements —
  trivial initialisation (MOVE ZEROS, MOVE SPACES) can be condensed
- Use `COMPUTE field = expression` for arithmetic
- Use `SELECT CASE field / WHEN value / END SELECT` for EVALUATE
- Use `-- annotation` for inline explanations of COBOL idioms
- Indent with 2 spaces per nesting level
- Represent 88-level condition names as their plain English equivalent
  where possible (e.g. `IF ACCT-OVERDRAWN` → `IF account balance is negative`)

---

## Statement translation catalogue

Translate each COBOL verb using these rules. Read source lines, identify
the verb, and apply the corresponding translation.

### MOVE

```cobol
MOVE WS-ACCT-ID TO DB-ACCT-ID
```
```pseudocode
SET db_account_id = ws_account_id
-- Transfer account ID to database record buffer
```

Special MOVE forms:
- `MOVE ZEROS TO field` → `CLEAR field  -- initialise to zero`
- `MOVE SPACES TO field` → `CLEAR field  -- initialise to spaces`
- `MOVE HIGH-VALUES TO field` → `SET field = MAX_VALUE  -- sentinel for end-of-file detection`
- `MOVE LOW-VALUES TO field` → `SET field = MIN_VALUE`
- `MOVE CORRESPONDING A TO B` → `COPY matching fields from A to B  -- COBOL CORRESPONDING: copies all fields with identical names`

### COMPUTE / ADD / SUBTRACT / MULTIPLY / DIVIDE

```cobol
COMPUTE WS-TOTAL = WS-AMOUNT * WS-RATE / 100
ADD 1 TO WS-COUNTER
SUBTRACT WS-DEBIT FROM WS-BALANCE
MULTIPLY WS-UNITS BY WS-PRICE GIVING WS-TOTAL
DIVIDE WS-TOTAL BY WS-COUNT GIVING WS-AVG REMAINDER WS-REM
```
```pseudocode
COMPUTE ws_total = ws_amount * ws_rate / 100
INCREMENT ws_counter BY 1
COMPUTE ws_balance = ws_balance - ws_debit
COMPUTE ws_total = ws_units * ws_price
COMPUTE ws_avg = ws_total / ws_count  -- remainder stored in ws_rem
```

Detect ROUNDED and ON SIZE ERROR clauses:
```cobol
COMPUTE WS-RESULT ROUNDED = WS-A + WS-B
  ON SIZE ERROR PERFORM 9900-OVERFLOW-HANDLER
```
```pseudocode
COMPUTE ws_result = ROUND(ws_a + ws_b)
IF arithmetic overflow THEN
  CALL 9900-overflow-handler
END IF
```

### IF / ELSE / END-IF

Translate conditions using the field catalogue for type context.
Expand 88-level condition names to their value meaning:

```cobol
IF ACCT-OVERDRAWN
   PERFORM 3000-APPLY-PENALTY
ELSE
   PERFORM 3100-PROCESS-NORMAL
END-IF
```
```pseudocode
-- ACCT-OVERDRAWN is true when ACCT-BALANCE is between -999999999.99 and -0.01
IF account_balance < 0 THEN
  CALL 3000-apply-penalty
ELSE
  CALL 3100-process-normal
END IF
```

Compound conditions:
- `AND` / `OR` → preserve as-is
- `NOT` → preserve as-is
- Abbreviated conditions (`IF A = 1 OR 2` → `IF A = 1 OR A = 2`) — expand

### EVALUATE

```cobol
EVALUATE WS-TRANS-TYPE
  WHEN 'CR'
    PERFORM 1000-PROCESS-CREDIT
  WHEN 'DR'
    PERFORM 1100-PROCESS-DEBIT
  WHEN OTHER
    PERFORM 9900-INVALID-TRANS
END-EVALUATE
```
```pseudocode
SELECT CASE transaction_type
  WHEN 'CR':
    -- Credit transaction
    CALL 1000-process-credit
  WHEN 'DR':
    -- Debit transaction
    CALL 1100-process-debit
  ELSE:
    -- Unrecognised transaction type
    CALL 9900-invalid-transaction
END SELECT
```

For `EVALUATE TRUE`:
```cobol
EVALUATE TRUE
  WHEN WS-AMT > 10000 AND WS-TYPE = 'LG'
    PERFORM 2000-LARGE-TRANS
  WHEN WS-AMT > 1000
    PERFORM 2100-MED-TRANS
  WHEN OTHER
    PERFORM 2200-SMALL-TRANS
END-EVALUATE
```
```pseudocode
IF amount > 10000 AND transaction_type = 'LG' THEN
  CALL 2000-large-transaction
ELSE IF amount > 1000 THEN
  CALL 2100-medium-transaction
ELSE
  CALL 2200-small-transaction
END IF
```

### READ

```cobol
READ ACCT-FILE INTO WS-ACCT-REC
  AT END MOVE 'Y' TO WS-EOF-FLAG
  NOT AT END PERFORM 0300-PROCESS-RECORD
END-READ
```
```pseudocode
READ next record from ACCT-FILE into account_record_buffer
IF end of file THEN
  SET end_of_file_flag = 'Y'
ELSE
  CALL 0300-process-record
END IF
```

Random READ (keyed):
```cobol
READ ACCT-FILE RECORD INTO WS-ACCT-REC
  KEY IS ACCT-ID
  INVALID KEY PERFORM 9100-NOT-FOUND
END-READ
```
```pseudocode
READ record from ACCT-FILE where key = account_id
IF record not found THEN
  CALL 9100-not-found
END IF
```

### WRITE / REWRITE / DELETE

```cobol
WRITE ACCT-RECORD FROM WS-ACCT-REC
  INVALID KEY PERFORM 9200-WRITE-ERROR
END-WRITE
```
```pseudocode
WRITE account_record to ACCT-FILE
IF write error (duplicate key or I/O failure) THEN
  CALL 9200-write-error
END IF
```

```cobol
REWRITE ACCT-RECORD FROM WS-ACCT-REC
DELETE ACCT-FILE RECORD
```
```pseudocode
UPDATE existing account_record in ACCT-FILE
DELETE current record from ACCT-FILE
```

### CALL

Static CALL:
```cobol
CALL 'CBACT04C' USING WS-COMM-AREA WS-RETURN-CODE
```
```pseudocode
CALL program CBACT04C
  PASSING: communication_area, return_code
  -- Subprogram: account validation routine
```

Dynamic CALL:
```cobol
CALL WS-PROG-NAME USING WS-PARM-AREA
```
```pseudocode
CALL program [dynamic: ws_prog_name]  -- target resolved at runtime
  PASSING: parameter_area
  !! REVIEW: dynamic call — target program unknown statically
```

### EXEC CICS commands

```cobol
EXEC CICS READ
  FILE('ACCTFILE')
  INTO(WS-ACCT-REC)
  RIDFLD(WS-ACCT-ID)
  RESP(WS-CICS-RESP)
END-EXEC
```
```pseudocode
CICS READ file ACCTFILE
  KEY: account_id
  INTO: account_record_buffer
  RESPONSE CODE → cics_response
IF cics_response NOT NORMAL THEN
  -- handle CICS error (see RESP handling below)
END IF
```

Common CICS commands:
| CICS command | Pseudocode |
|---|---|
| `EXEC CICS READ` | `CICS READ file KEY=... INTO=...` |
| `EXEC CICS WRITE` | `CICS WRITE file FROM=... RIDFLD=...` |
| `EXEC CICS REWRITE` | `CICS UPDATE file FROM=...` |
| `EXEC CICS DELETE` | `CICS DELETE file KEY=...` |
| `EXEC CICS SEND MAP` | `CICS SEND screen map=... TO terminal` |
| `EXEC CICS RECEIVE MAP` | `CICS RECEIVE input from screen map=...` |
| `EXEC CICS LINK PROGRAM` | `CICS CALL program=... PASSING commarea` |
| `EXEC CICS XCTL PROGRAM` | `CICS TRANSFER CONTROL TO program=... (no return)` |
| `EXEC CICS RETURN` | `CICS RETURN TO caller` |
| `EXEC CICS RETURN TRANSID` | `CICS RETURN, next transaction = transid` |
| `EXEC CICS GETMAIN` | `CICS ALLOCATE memory BYTES=... INTO=...` |
| `EXEC CICS ABEND` | `CICS ABEND with code=...  -- terminates transaction` |

### EXEC SQL

```cobol
EXEC SQL
  SELECT ACCT_BAL INTO :WS-BALANCE
  FROM ACCOUNT
  WHERE ACCT_ID = :WS-ACCT-ID
END-EXEC
```
```pseudocode
SQL SELECT account_balance
  FROM: ACCOUNT table
  WHERE: account_id = ws_account_id
  INTO: ws_balance
CHECK SQLCODE for errors
```

Common SQL patterns:
| SQL pattern | Pseudocode |
|---|---|
| `SELECT ... INTO` | `SQL SELECT field(s) from table WHERE condition INTO host variable(s)` |
| `INSERT INTO` | `SQL INSERT new row INTO table` |
| `UPDATE ... SET` | `SQL UPDATE table SET field = value WHERE condition` |
| `DELETE FROM` | `SQL DELETE from table WHERE condition` |
| `DECLARE CURSOR` | `DECLARE SQL CURSOR for query` |
| `OPEN CURSOR` | `OPEN cursor and execute query` |
| `FETCH ... INTO` | `SQL FETCH next row from cursor INTO host variables` |
| `CLOSE CURSOR` | `CLOSE cursor` |

Always append: `CHECK SQLCODE` — DB2 programs check SQLCODE after every
SQL statement. Record the SQLCODE check logic as a branch.

### STRING / UNSTRING

```cobol
STRING WS-FIRST-NAME DELIMITED SPACE
       '-'            DELIMITED SIZE
       WS-LAST-NAME  DELIMITED SPACE
  INTO WS-FULL-NAME
```
```pseudocode
CONCATENATE first_name + '-' + last_name INTO full_name
-- COBOL STRING: joins fields, stopping at delimiter (SPACE stops at first space)
```

```cobol
UNSTRING WS-DATE-STRING DELIMITED '/'
  INTO WS-YEAR WS-MONTH WS-DAY
```
```pseudocode
SPLIT date_string BY '/' INTO year, month, day
-- COBOL UNSTRING: splits a string on a delimiter
```

### INSPECT

```cobol
INSPECT WS-FIELD TALLYING WS-COUNT FOR ALL 'A'
INSPECT WS-FIELD REPLACING ALL SPACES BY ZEROS
```
```pseudocode
COUNT occurrences of 'A' in field → store in ws_count
REPLACE all spaces in field with zeros
-- COBOL INSPECT: character-level scan and replace
```

### OPEN / CLOSE

```cobol
OPEN INPUT ACCT-FILE
OPEN I-O TRANS-FILE
OPEN OUTPUT REPORT-FILE
CLOSE ACCT-FILE
```
```pseudocode
OPEN ACCT-FILE for READ-ONLY
OPEN TRANS-FILE for READ-WRITE
OPEN REPORT-FILE for WRITE (new)
CLOSE ACCT-FILE
```

### STOP RUN / GOBACK / EXIT PROGRAM

```pseudocode
STOP RUN           → TERMINATE program
GOBACK             → RETURN to caller (or terminate if main program)
EXIT PROGRAM       → RETURN to caller
EXIT               → EXIT current paragraph (no-op if inline)
```

---

## Phase — program narrative summary

After translating all paragraphs, generate a short program-level narrative
(3–8 sentences) that describes what the program does at a business level.
Use the pseudocode blocks as source material. Write in plain English.

Template:
```
Program {PROGRAM-ID} is a {batch/online} program that {primary purpose}.
It reads from {files/tables}, processes {what}, and writes to {files/tables}.
The main loop {describes loop behaviour}.
Key business operations include: {list top 3-5 operations from pseudocode}.
Error conditions handled: {list error paths}.
```

---

## Output schema — per-paragraph pseudocode block

```json
{
  "program_id": "CBACT01C",
  "program_narrative": "CBACT01C is a batch program that processes account transactions from ACCT-FILE. It reads each transaction record sequentially, validates the account ID against the account master file, applies credits or debits based on the transaction type, and writes updated balances to the account master. Error conditions include invalid account IDs, overdraft detection, and I/O failures on both input and output files.",
  "paragraphs": [
    {
      "name": "0000-MAIN",
      "section": "0000-MAIN",
      "line_range": [425, 480],
      "pseudocode": [
        "-- Main control paragraph: orchestrates the full batch run",
        "CALL 0100-open-files",
        "CALL 0150-initialise-counters",
        "READ first record from TRANS-FILE",
        "WHILE end_of_file_flag != 'Y' DO",
        "  CALL 0200-process-transaction",
        "  READ next record from TRANS-FILE",
        "END WHILE",
        "CALL 9000-close-files",
        "CALL 9100-print-totals",
        "TERMINATE program"
      ],
      "pseudocode_text": "CALL 0100-open-files\nCALL 0150-initialise-counters\n...",
      "field_references": ["WS-EOF-FLAG", "WS-TRANS-REC"],
      "calls_made": ["0100-OPEN-FILES", "0200-PROCESS-TRANSACTION"],
      "complexity_score": 2,
      "has_unresolved_statements": false,
      "annotations": []
    },
    {
      "name": "0200-PROCESS-TRANSACTION",
      "section": "0000-MAIN",
      "line_range": [520, 610],
      "pseudocode": [
        "-- Process a single transaction record",
        "SET db_account_id = trans_account_id",
        "CICS READ file ACCTFILE KEY: account_id INTO: account_record_buffer",
        "  RESPONSE CODE → cics_response",
        "IF cics_response = NORMAL THEN",
        "  SELECT CASE transaction_type",
        "    WHEN 'CR':",
        "      COMPUTE account_balance = account_balance + transaction_amount",
        "      INCREMENT credit_count BY 1",
        "    WHEN 'DR':",
        "      IF account_balance >= transaction_amount THEN",
        "        COMPUTE account_balance = account_balance - transaction_amount",
        "        INCREMENT debit_count BY 1",
        "      ELSE",
        "        CALL 8000-insufficient-funds",
        "      END IF",
        "    ELSE:",
        "      CALL 9900-invalid-transaction-type",
        "  END SELECT",
        "  CICS UPDATE file ACCTFILE FROM: account_record_buffer",
        "ELSE",
        "  CALL 9500-account-not-found",
        "END IF"
      ],
      "field_references": [
        "WS-ACCT-ID", "WS-TRANS-TYPE", "WS-AMOUNT",
        "ACCT-BALANCE", "WS-CICS-RESP"
      ],
      "calls_made": [
        "8000-INSUFFICIENT-FUNDS",
        "9500-ACCOUNT-NOT-FOUND",
        "9900-INVALID-TRANSACTION-TYPE"
      ],
      "complexity_score": 6,
      "has_unresolved_statements": false,
      "annotations": [
        {
          "line": 545,
          "note": "CICS READ used instead of file READ — this is an online program accessing VSAM via CICS"
        }
      ]
    }
  ],
  "unresolved_statements": [],
  "pseudocode_issues": []
}
```

### Complexity score

Assign a complexity score (1–10) to each paragraph based on:
- +1 per IF/ELSE block
- +1 per EVALUATE clause
- +2 per nested IF (depth > 1)
- +1 per PERFORM UNTIL or PERFORM VARYING
- +2 per GO TO
- +3 per ALTER
- +1 per CALL
- +1 per I/O operation (READ/WRITE/CICS/SQL)

Score interpretation:
- 1–3: Simple — straightforward sequential logic
- 4–6: Moderate — some branching, manageable
- 7–9: Complex — heavy branching, needs careful review
- 10+: Critical — flag for mandatory SME review

---

## Error handling

| Condition | Action |
|---|---|
| Statement verb not in translation catalogue | Record raw COBOL verbatim, wrap in `!! UNTRANSLATED:`, log `warning` |
| Field reference not in field_catalogue | Use field name as-is in pseudocode, log `info` |
| 88-level condition value cannot be expressed plainly | Use condition name as-is with `-- see data dictionary` annotation |
| CICS command not in translation table | Record as `CICS {VERB} [parameters]` and log `info` |
| SQL too complex to summarise (subqueries, joins) | Summarise intent, append `-- see source line {N} for full SQL` |
| Dynamic CALL target unknown | Use `[dynamic: variable-name]` placeholder, flag `!! REVIEW` |
| Paragraph body is entirely COPY-expanded | Note `-- body defined in copybook {NAME}` and flag for Data Agent |
