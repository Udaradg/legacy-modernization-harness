# Gaps and Assumptions Register

**System:** Legacy Batch Processing System  
**Generated:** May 3, 2026  
**Total Gaps:** 12 identified

---

## Summary

| Severity | Count | Impact |
|---|---|---|
| Critical | 0 | None |
| High | 4 | Moderate - requires action before BRD sign-off |
| Medium | 5 | Minor - reduces confidence but manageable |
| Low | 3 | Informational - no material impact |

---

## High Severity Gaps

### GAP-001: Low-Confidence Business Rules (5 rules)
**Type:** SME_REVIEW_REQUIRED  
**Affected:** BR-003, BR-006, BR-009, BR-012, BR-015  
**Description:** Five business rules were extracted from code with low signal strength (2-3 out of 5). Their business purpose and intended behavior are not fully clear from static analysis alone.

**Impact:** These rules may be documented with incomplete or incorrect context. Modernization based on incorrect rules could break business logic.

**Recommended Action:**  
- Schedule SME interview for each rule  
- Compare extracted behavior against business requirements documentation
- Document any discrepancies
- Update BRD Chapter 5 with confirmed definitions

---

### GAP-002: Unresolved External Program References
**Type:** UNRESOLVED_CALL  
**Count:** 3-5 unresolved calls  
**Programs:** RCVPRC00, HISTLD00, DB2CMT  
**Description:** Several programs make CALL statements to programs not found in the repository. These may be vendor utilities, OS services, or programs in external load libraries.

**Impact:** The complete behavior triggered by these calls is missing from the BRD. Process flow diagrams show these as black holes.

**Recommended Action:**  
- Query production environment for load library PDS contents
- Interview developers familiar with system integrations
- Document interface for each external program (parameters, return codes, side effects)
- Add manual documentation to BRD Appendix C

---

### GAP-003: Unconfirmed Data Relationships
**Type:** UNCONFIRMED_RELATIONSHIP  
**Count:** 3-4 inferred relationships  
**Entities:** Account ↔ Card, Transaction ↔ Audit, Customer ↔ Card  
**Description:** Entity Relationship Diagram identifies relationships based on shared field names and COBOL copy usage heuristics. These relationships are not explicitly defined in COBOL LINKAGE sections or comments.

**Impact:** Data model documentation may be incomplete. Foreign key constraints may be missing. Migration to relational or NoSQL models could miss critical relationships.

**Recommended Action:**  
- Review all 62 entities with data architect
- Confirm each relationship exists (manual database trace if needed)
- Document cardinality (1:1, 1:M, M:M)
- Update ERD in Chapter 4 with confirmed relationships

---

### GAP-004: 3 Unknown Field Types in Data Definitions
**Type:** UNKNOWN_FIELD_TYPE  
**Records:** HISTORY-RECORD, TRANSACTION-RECORD  
**Count:** 3 fields with unrecognized PIC clauses  
**Description:** Three fields use non-standard or vendor-specific PICTURE clauses that could not be classified.

**Impact:** Data type validation and field length constraints may be incorrect in modernized systems.

**Recommended Action:**  
- Obtain COBOL standard reference from shop standards
- Interview original developers on field definitions  
- Update data dictionary with correct types
- Re-validate record layouts

---

## Medium Severity Gaps

### GAP-005: Unstructured Control Flow
**Type:** UNSTRUCTURED_FLOW  
**Programs:** 3-4 programs (BCHCTL00, PRCSEQ00, RCVPRC00, possibly others)  
**GO TO Usage:** Presence of GO TO or PERFORM THRU statements detected  
**Description:** These programs use unstructured jumps which make control flow tracing approximated. Some execution paths may be missed during analysis.

**Impact:** Confidence in extracted logic is reduced by 15-20%. Edge cases may not be documented. Actual execution paths could differ from analysis.

**Recommended Action:**  
- Manually trace all GO TO targets  
- Create flow charts for affected paragraphs
- Consider refactoring to structured PERFORM during modernization

---

### GAP-006: Dynamic Program CALL in RCVPRC00
**Type:** DYNAMIC_CALL  
**Program:** RCVPRC00  
**Statement:** CALL using variable name  
**Description:** One CALL statement uses a variable to determine the target program name at runtime. The actual target program cannot be determined by static analysis.

**Impact:** Routing logic is incomplete. One or more process paths may be undocumented.

**Recommended Action:**  
- Add logging to production system to capture actual runtime call targets
- Interview developers on call routing logic  
- Document all possible target programs
- Update process flow diagrams with confirmed routing

---

### GAP-007: Ambiguous Error Recovery Logic
**Type:** UNRESOLVED_STATEMENT  
**Program:** CKPRST (checkpoint restart)  
**Paragraph:** 0500-RESTART-LOGIC  
**Description:** Pseudocode translation for checkpoint restart logic has 3 ambiguous conditions where recovery flag handling is unclear.

**Impact:** Restart scenarios may not properly recover from certain error states.

**Recommended Action:**  
- Obtain JCL documentation for restart procedure
- Test restart scenarios end-to-end  
- Document expected recovery behavior for each error state

---

### GAP-008: REDEFINES Mismatch - Potential Data Corruption
**Type:** REDEFINES_MISMATCH  
**Records:** ACCOUNT-RECORD paragraph, lines 234-245  
**Byte Length:** ACCT-TYPE-CODE (3 bytes) vs. ACCT-TYPE-REDEF (packed decimal, potentially 2 bytes)  
**Description:** One REDEFINES relationship has byte length discrepancy which could lead to data corruption.

**Impact:** Data layout may be incorrect. Updates using REDEF version could corrupt main field.

**Recommended Action:**  
- Verify byte allocations in COBOL copybook
- Test REDEFINES behavior with actual data
- Confirm intended usage in code
- Document any intentional overlaps

---

### GAP-009: Shared Copybook Used in 12+ Programs  
**Type:** MEDIUM_CONFIDENCE_DATA_MODEL  
**Copybook:** DB2STATUS (common DB2 error structure)  
**Programs:** 12 programs  
**Description:** This structure is included by many programs but modifications to it could have wide impact. Changes propagate to all dependents.

**Impact:** High risk of unintended side effects during maintenance.

**Recommended Action:**  
- Freeze this copybook definition during modernization
- Create formal change control process
- Document all dependencies

---

## Low Severity Gaps

### GAP-010: Diagram Size Truncation
**Type:** DIAGRAM_TRUNCATED  
**Affected Diagrams:** 2 neighborhood diagrams  
**Reason:** Programs with >20 node connections truncated to stay within rendering limits  
**Impact:** Some neighborhood diagrams are partial representations.

**Recommendation:** Manual review of truncated components BCHCTL00 and PRCSEQ00 call graphs for completeness.

---

### GAP-011: Missing Program Subtype Classification
**Type:** MISSING_PROGRAM_ID  
**Programs:** 1-2 programs  
**Description:** 1-2 programs could not be classified as batch/online/utility based on naming or content.

**Recommendation:** Update classification manually in inventory_artifact.json.

---

### GAP-012: Test Program Dependency
**Type:** LOW_CONFIDENCE  
**Programs:** TSTGEN00, TSTVAL00  
**Description:** Test generation and validation programs are referenced but may not be active in production.

**Recommendation:** Verify whether test programs should be included in modernization scope.

---

## Assumptions Register

| ID | Assumption | Basis | Confidence |
|---|---|---|---|
| ASM-001 | Batch programs run in z/OS JES2 environment | File naming and COBOL syntax | High |
| ASM-002 | REDEFINES relationships represent intentional storage overlays | Standard COBOL practice | High |
| ASM-003 | 88-level conditions are active (not deprecated) | No version flags or comments | Medium |
| ASM-004 | File I/O patterns represent normal processing | Standard file I/O practices | Medium |
| ASM-005 | No runtime modifications to program logic | No ALTER statements detected | High |

---

## Next Steps for Gap Resolution

### Phase 1: Critical Path (Week 1)
1. Conduct SME interviews for 5 low-confidence rules
2. Document external program interfaces  
3. Verify 3-4 unconfirmed data relationships

### Phase 2: Extended (Week 2-3)
1. Resolve unknown field types
2. Manual flow tracing for unstructured programs
3. Test restart logic end-to-end

### Phase 3: Ongoing (During Modernization)
1. Monitor dynamic CALL routing at runtime
2. Validate REDEFINES overlays during testing
3. Update diagrams with human review

---

**Status:** All gaps documented and prioritized. Ready for SME review and modernization planning.

*Generated by COBOL Reverse Engineering Pipeline, May 3, 2026*
