# Gaps and assumptions register

**System:** Portfolio Management System
**Generated:** 2026-05-03T14:50:00Z
**Pipeline version:** agents 1–7 @1.0
**Total gaps:** 14 (0 critical, 2 high, 5 medium, 7 low)

---

## Critical gaps — none identified

No critical gaps were found. There are no ALTER statements, no parse failures, no unresolved copybooks, no circular copy dependencies, and no confirmed infinite loops in the codebase. The pipeline was able to analyse all 38 programs successfully.

---

## High severity gaps

### GAP-001 — UTLMON00 calls ILBOABN0 — external abnormal-termination service not in repository

**Type:** UNRESOLVED_CALL | **Program:** UTLMON00 | **Line:** 186

Program UTLMON00 (utility monitoring) makes a static CALL to ILBOABN0, which is not present in the repository. ILBOABN0 is a standard IBM z/OS Language Environment routine that triggers a controlled abnormal program termination (abend). Its behaviour is well-understood in the IBM mainframe context, but it is not documented by the pipeline.

**Impact:** The termination path in UTLMON00 is partially undocumented in the BRD. The program's response to irrecoverable monitoring errors is known by convention (program abend) but not confirmed by static analysis.

**Recommended action:** Confirm ILBOABN0 is the IBM LE abend routine. Document the abend code it generates and whether any upstream job-control or restart logic intercepts it.

---

### GAP-002 — DB2CONN calls DELAY — external timing service not in repository

**Type:** UNRESOLVED_CALL | **Program:** DB2CONN | **Line:** 86

Program DB2CONN (DB2 connection manager) calls DELAY inside its connection retry loop. DELAY is not present in the repository. It is believed to be an IBM system timing service used to introduce a pause between connection retry attempts. The retry count limit is enforced by business rule RULE-ENFORCE-WS-RETRY-COUNT-HAS-REQUIRED-VALUE.

**Impact:** The retry-wait logic in DB2CONN is incomplete. The delay duration between retry attempts cannot be determined without knowing the DELAY routine interface.

**Recommended action:** Identify the DELAY routine. Document its calling interface, the delay duration passed, and confirm whether it is configurable at runtime.

---

## Medium severity gaps

### GAP-003 — SQL include SQLPOS not found

**Type:** UNRESOLVED_REFERENCE | **Programs:** INQPORT, INQHIST | **Line:** ~18

Programs INQPORT and INQHIST reference EXEC SQL INCLUDE SQLPOS, but SQLPOS is not present in the copybook registry. This is typically a DB2-generated host-variable structure produced at pre-compile time.

**Impact:** The field layout of position data returned by SQL queries in INQPORT and INQHIST cannot be fully documented. Position inquiry field names, types, and lengths are incomplete.

**Recommended action:** Locate SQLPOS in the DB2 pre-compile output or DCLGEN library. Add to src/copybook/db2/ and re-run the Data Agent.

---

### GAP-004 — POSUPDT source file is empty

**Type:** EMPTY_FILE | **Program:** POSUPDT

Program POSUPDT (position update) is registered in the inventory but its source file contains only whitespace. The program has zero paragraphs and zero lines of logic.

**Impact:** Any batch process relying on POSUPDT to update investment positions is completely undocumented. This gap is especially significant given that PORTTRAN's position-update paragraph is also flagged as dead code (see GAP-007).

**Recommended action:** Investigate whether POSUPDT exists in a load library or different source repository. If it is an unimplemented stub, document the intended functionality.

---

### GAP-005 — Unconfirmed data relationship: Error Handling → Return Handling via ERROR-CODE

**Type:** UNCONFIRMED_RELATIONSHIP | **Source:** data_artifact REL-001

The data model contains a heuristic-detected relationship between ENT-ERROR-HANDLING and ENT-RETURN-HANDLING based on the shared field ERROR-CODE. Confidence is medium.

**Impact:** The ERD in Chapter 4 shows this relationship but marks it as unconfirmed. If the relationship is not genuine, the ERD implies a data coupling that does not exist in practice.

**Recommended action:** Review with a data architect or DBA. Confirm whether records in one structure genuinely reference records in the other.

---

### GAP-006 — INQONLN contains 6 confirmed unreachable exit paragraphs

**Type:** DEAD_CODE_CONFIRMED | **Program:** INQONLN

Paragraphs P100-EXIT, P200-EXIT, P300-EXIT, P400-EXIT, P900-EXIT, and P050-EXIT in INQONLN are confirmed unreachable. These are conventional PERFORM THRU exit markers with no business logic content.

**Impact:** No business functionality is missing. However, their presence may cause confusion during modernisation if they are mistaken for active logic.

**Recommended action:** Confirm with the development team that these are intentional PERFORM THRU exit points. If not used with PERFORM THRU, document them as safely removable.

---

### GAP-007 — PORTTRAN paragraph 2200-UPDATE-POSITIONS is confirmed unreachable — potential defect

**Type:** DEAD_CODE_CONFIRMED | **Program:** PORTTRAN

The Logic Agent confirmed that paragraph 2200-UPDATE-POSITIONS in PORTTRAN is never called during normal execution. This paragraph's name suggests it updates investment position quantities — a critical step in transaction processing.

**Impact:** If 2200-UPDATE-POSITIONS was intended to be active logic, transaction processing may not be correctly updating position quantities. This is the most operationally significant dead code finding in the codebase. Note that POSUPDT (position update program) is also empty (see GAP-004), compounding this concern.

**Recommended action:** Urgently review PORTTRAN with a business analyst and developer. Confirm whether position updates are handled elsewhere (DB2 trigger, separate program call) or whether this is a genuine defect.

---

## Low severity gaps

### GAP-008 — PORTMSTR paragraph 2100-HANDLE-VSAM-ERROR is confirmed unreachable

**Type:** DEAD_CODE_CONFIRMED | **Program:** PORTMSTR

Paragraph 2100-HANDLE-VSAM-ERROR in PORTMSTR is confirmed unreachable. This is a VSAM file error handler for portfolio file operations.

**Impact:** VSAM errors on portfolio file operations may be handled by a generic error handler rather than the specific VSAM handler, potentially losing file-status context.

**Recommended action:** Review PORTMSTR to confirm how VSAM errors are currently handled. Confirm whether the generic handler is adequate.

---

### GAP-009 — RTNANA00 and RTNCDE00 each have 11 confirmed dead code candidates

**Type:** DEAD_CODE_CONFIRMED | **Programs:** RTNANA00, RTNCDE00

The Parser Agent identified 11 dead code candidates in each of RTNANA00 and RTNCDE00 — the highest counts per program in the codebase.

**Impact:** Portions of return code analysis and processing logic may be unreachable, possibly representing legacy code paths.

**Recommended action:** Review candidates with the development team. Determine whether they represent legacy logic (safe to remove) or untested paths (potentially valuable).

---

### GAP-010 — Two similarly-named error copybooks ERRHAND and ERRHND may be confused

**Type:** DUPLICATE_IDENTIFIER | **Source:** inventory_artifact issues

ERRHAND (batch error handling, 6 programs) and ERRHND (CICS online error handling) have near-identical names.

**Impact:** Developers may accidentally include the wrong copybook. The two structures serve different purposes but their similarity may not be obvious.

**Recommended action:** Consider renaming one copybook during modernisation (e.g., ERRHAND-BATCH and ERRHAND-CICS).

---

### GAP-011 — SQLCA included as both EXEC SQL INCLUDE and COPY in two programs

**Type:** DUPLICATE_IDENTIFIER | **Programs:** RTNCDE00, CURSMGR

Programs RTNCDE00 and CURSMGR include SQLCA using both EXEC SQL INCLUDE and COPY statements.

**Impact:** Pre-compile errors may occur, or one inclusion may silently override the other.

**Recommended action:** Remove the duplicate inclusion. Prefer EXEC SQL INCLUDE SQLCA as the standard DB2 mechanism.

---

### GAP-012 — 79 of 90 business rules extracted at medium confidence

**Type:** MEDIUM_CONFIDENCE_RULE | **Source:** rules_artifact

Of 90 business rules extracted, 79 are medium confidence. They correctly describe code behaviour but may not capture full business intent.

**Impact:** Rules catalogue entries should be treated as preliminary findings pending expert validation.

**Recommended action:** Review medium-confidence rules with domain experts. Focus on PORTTRAN validation rules, PRCSEQ00 sequencing rules, and PORTMSTR portfolio validation rules.

---

### GAP-013 — COMMON copybook defined but not used by any program

**Type:** UNRESOLVED_REFERENCE | **Source:** inventory_artifact copybook_registry

The COMMON copybook (src/copybook/common/COMMON.cpy) defines shared data structures for return codes, status codes, transaction types, date/time, error handling, audit fields, and currency codes — but no program includes it.

**Impact:** Programs may define their own local versions of common data structures, potentially leading to inconsistencies.

**Recommended action:** Determine whether COMMON is used by programs not currently in the repository, or was never adopted. If it represents the intended standard, consider whether programs should be updated to use it.

---

### GAP-014 — 15 programs have 'unknown' execution subtype

**Type:** UNRESOLVED_REFERENCE | **Source:** inventory_artifact file_registry

Portfolio programs (PORTADD, PORTDEL, PORTMSTR, PORTREAD, PORTTEST, PORTTRAN, PORTUPDT, PORTVALD) and common programs (AUDPROC, DB2CMT, DB2CONN, DB2ERR, DB2STAT, ERRPROC) have subtype 'unknown'. They could not be positively identified as batch or online from source code alone.

**Impact:** The program inventory cannot accurately classify these programs by execution mode, affecting architectural analysis.

**Recommended action:** Review each program's PROCEDURE DIVISION and calling contexts. Portfolio programs are most likely called sub-programs invocable from both batch and CICS contexts.

---

## Assumptions

| ID | Description | Basis | Confidence |
|---|---|---|---|
| ASM-001 | The system is a financial portfolio management system supporting investment account management, transaction processing, online inquiry, and batch reporting on IBM z/OS. | Program/field/copybook names and Logic Agent narratives | High |
| ASM-002 | Portfolio programs are called sub-programs invocable from both batch JCL and CICS transactions. | Programs receive parameters via LINKAGE SECTION with no explicit CICS or batch-specific markers | Medium |
| ASM-003 | ERRPROC is the central error logging sub-program, called by at least 8 programs. | inventory_artifact call_graph — ERRPROC has most incoming call edges | High |
| ASM-004 | Portfolio master file is a VSAM KSDS keyed on PORT-ID with a 'PORT' prefix format requirement. | data_artifact PORT-RECORD structure; PORTMSTR validation PORT-ID(1:4) NOT = 'PORT' | High |
| ASM-005 | DB2 stores historical data, audit logs, and statistics. VSAM stores current portfolio records and batch control data. | Program names DB2CMT, HISTLD00 and VSAM file status field patterns in batch programs | Medium |
| ASM-006 | Transaction types BUY, SELL, and TRANSFER (code 'TR') are the three supported investment transaction types. | PORTTRAN EVALUATE on TRN-TYPE; TRN-TYPE NOT = 'TR' exception in price/amount validation | High |
| ASM-007 | BCHCTL represents a shared batch job control record tracking execution status, return codes, and sequencing. | PRCSEQ00 reads BCT-JOB-NAME, BCT-STAT-READY, BCT-STAT-ACTIVE, BCT-STAT-ERROR, BCT-STAT-DONE | High |
