---
name: gap-detector
description: >
  Scans all six prior artifacts for every unresolved reference, low-confidence
  rule, dynamic call, dead code block, missing copybook, ambiguous pseudocode
  statement, and SME review flag. Consolidates them into a prioritised gaps
  and assumptions register with severity levels, source traceability, and
  recommended resolution actions. Produces gaps_register.json and
  gaps_register.md. Called by the Synthesis Agent before section-assembler
  so that gap annotations are available during BRD writing.
---

# Skill — gap detector

## Purpose

Before the BRD is written, perform a full cross-artifact audit. Collect
every signal of incompleteness, uncertainty, or ambiguity that the pipeline
produced. Classify each by severity and type, deduplicate where the same
root cause appears in multiple artifacts, and produce a prioritised register
that tells SMEs exactly what needs human validation.

---

## Gap source catalogue

Scan the following locations across all artifacts:

### From inventory_artifact.json

| Field | Gap type |
|---|---|
| `call_graph.edges` where `resolved: false` | `UNRESOLVED_CALL` |
| `call_graph.edges` where `type: DYNAMIC_CALL` | `DYNAMIC_CALL` |
| `issues[]` where `type: unresolved_reference` | `UNRESOLVED_REFERENCE` |
| `issues[]` where `type: circular_copy` | `CIRCULAR_COPY` |
| `issues[]` where `type: missing_program_id` | `MISSING_PROGRAM_ID` |

### From parser_artifact.json

| Field | Gap type |
|---|---|
| `programs[]` where `parse_status: failed` | `PARSE_FAILURE` |
| `programs[]` where `has_unstructured_flow: true` | `UNSTRUCTURED_FLOW` |
| `programs[]` where `has_alter: true` | `ALTER_STATEMENT` — critical |
| `stats.total_goto_edges > 0` | `GOTO_USAGE` |
| `stats.total_dead_code_candidates > 0` | `DEAD_CODE` |

### From data_artifact.json

| Field | Gap type |
|---|---|
| `issues[]` where `type: unresolved_copy_stub` | `UNRESOLVED_COPYBOOK` |
| `field_catalogue[]` where `normalized_type: UNKNOWN` | `UNKNOWN_FIELD_TYPE` |
| `data_model.relationships[]` where `confidence: low` | `UNCONFIRMED_RELATIONSHIP` |
| `redefines_groups[]` with mismatched byte lengths | `REDEFINES_MISMATCH` |

### From logic_artifact.json

| Field | Gap type |
|---|---|
| `programs[].unresolvable_paths[]` | `UNRESOLVABLE_PATH` |
| `paragraphs[]` where `has_unresolved_statements: true` | `UNRESOLVED_STATEMENT` |
| `loops[]` where `termination_pattern: POTENTIAL_INFINITE_LOOP` | `INFINITE_LOOP_RISK` |
| `dead_code_confirmed[]` | `DEAD_CODE_CONFIRMED` |
| `pseudocode_issues[]` | `PSEUDOCODE_ISSUE` |

### From rules_artifact.json

| Field | Gap type |
|---|---|
| `business_rules[]` where `requires_sme_review: true` | `SME_REVIEW_REQUIRED` |
| `business_rules[]` where `confidence: low` | `LOW_CONFIDENCE_RULE` |
| `business_rules[]` where `confidence: medium` | `MEDIUM_CONFIDENCE_RULE` |
| `issues[]` where `type: sme_review_required` | `SME_REVIEW_REQUIRED` |

### From diagrams_artifact.json

| Field | Gap type |
|---|---|
| Any diagram where `truncated: true` | `DIAGRAM_TRUNCATED` |
| Process flows with `dead_code_nodes > 0` | `DEAD_CODE` (cross-reference) |

---

## Severity classification

Assign severity to each gap based on its type and impact:

### Critical — must resolve before BRD can be considered complete

| Gap type | Reason |
|---|---|
| `ALTER_STATEMENT` | ALTER changes GO TO targets at runtime — logic is fundamentally unknowable without runtime traces |
| `PARSE_FAILURE` | Program was not parsed — its rules, data, and logic are absent from the BRD |
| `UNRESOLVED_COPYBOOK` | Data definitions are incomplete — field types and layouts may be wrong |
| `INFINITE_LOOP_RISK` | Program may never terminate — critical operational risk |
| `CIRCULAR_COPY` | Copybook dependency cycle — data definitions may be incorrect |

### High — significantly affects BRD completeness or accuracy

| Gap type | Reason |
|---|---|
| `UNRESOLVED_CALL` | Called program's behaviour is unknown — process flow has a black hole |
| `DYNAMIC_CALL` | Target program unknown at analysis time — routing logic incomplete |
| `SME_REVIEW_REQUIRED` | Rule extracted but purpose unclear — may be wrong or incomplete |
| `UNRESOLVABLE_PATH` | Execution path could not be traced — process flow may be incomplete |
| `REDEFINES_MISMATCH` | Data layout is potentially corrupt — field offsets may be wrong |

### Medium — reduces confidence but BRD is still usable

| Gap type | Reason |
|---|---|
| `UNSTRUCTURED_FLOW` | GO TO usage makes flow tracing approximated — some paths may be missed |
| `GOTO_USAGE` | Unstructured jumps increase risk of missed logic |
| `LOW_CONFIDENCE_RULE` | Business rule inferred but not confirmed — needs validation |
| `UNCONFIRMED_RELATIONSHIP` | Data relationship detected by heuristic — needs data model review |
| `UNRESOLVED_STATEMENT` | Pseudocode has untranslated COBOL — section may be incomplete |

### Low — minor gaps, BRD accuracy not materially affected

| Gap type | Reason |
|---|---|
| `DEAD_CODE_CONFIRMED` | Unreachable code exists — may represent retired functionality |
| `MEDIUM_CONFIDENCE_RULE` | Rule is likely correct but warrants confirmation |
| `UNKNOWN_FIELD_TYPE` | One or more fields have unrecognised PIC clause |
| `MISSING_PROGRAM_ID` | Program ID inferred from filename — may differ from intended ID |
| `DIAGRAM_TRUNCATED` | Some diagrams are partial due to node count limits |
| `PSEUDOCODE_ISSUE` | Minor translation issue in pseudocode — source is available |

---

## Deduplication

The same root cause may appear in multiple artifacts. Before producing
the register, deduplicate:

```
1. Group gaps by (gap_type + source_program + source_field/paragraph)
2. If two gaps share the same root (e.g. UNRESOLVED_CALL to same target
   appearing in both inventory and logic artifacts) → merge into one entry
3. List all artifact sources in the merged entry's `detected_in` field
4. Use the highest severity from the merged set
```

---

## Recommended resolution actions

For each gap type, include a standard recommended action:

| Gap type | Recommended action |
|---|---|
| `ALTER_STATEMENT` | Obtain runtime execution traces or interview original developers. Document all possible GO TO targets for each ALTER statement. |
| `PARSE_FAILURE` | Manually review the source file. Check for non-standard COBOL dialect, free-format indicators, or encoding issues. |
| `UNRESOLVED_COPYBOOK` | Locate the missing copybook in a load library, external PDS, or vendor-supplied library. Add to repo and re-run Data Agent. |
| `UNRESOLVED_CALL` | Determine if target is in a load library, vendor package, or OS utility. Document the interface and expected behaviour. |
| `DYNAMIC_CALL` | Add logging to the production system to capture actual runtime values of the call variable, or interview developers. |
| `INFINITE_LOOP_RISK` | Review the loop termination field — confirm it is always set within the loop body. Check for missing READ AT END or MOVE to flag field. |
| `SME_REVIEW_REQUIRED` | Present the extracted rule description to a subject matter expert for confirmation or correction. |
| `LOW_CONFIDENCE_RULE` | Review the pseudocode context for this rule and confirm with a business analyst or developer familiar with the domain. |
| `UNSTRUCTURED_FLOW` | Map GO TO targets manually. Consider refactoring GO TO chains into structured PERFORM before modernisation. |
| `DEAD_CODE_CONFIRMED` | Confirm with development team whether the code is intentionally retired or accidentally unreachable. |
| `UNCONFIRMED_RELATIONSHIP` | Validate with data architect or DBA whether the shared key field represents a genuine foreign key relationship. |

---

## Output schema — gaps_register.json

```json
{
  "meta": {
    "generated_at": "ISO-8601 timestamp",
    "agent_version": "7_synthesis@1.0",
    "total_gaps": 48,
    "by_severity": {
      "critical": 5,
      "high": 12,
      "medium": 18,
      "low": 13
    },
    "by_type": {
      "SME_REVIEW_REQUIRED": 18,
      "UNRESOLVED_CALL": 8,
      "LOW_CONFIDENCE_RULE": 7,
      "UNSTRUCTURED_FLOW": 6,
      "DEAD_CODE_CONFIRMED": 5,
      "DYNAMIC_CALL": 4
    }
  },
  "gaps": [
    {
      "gap_id": "GAP-001",
      "gap_type": "ALTER_STATEMENT",
      "severity": "critical",
      "title": "ALTER statement in CBACT01C modifies GO TO target at runtime",
      "description": "Program CBACT01C contains an ALTER statement in paragraph 0500-ROUTE that modifies the GO TO target of paragraph 0510-DISPATCH at runtime based on transaction type. The actual target paragraph cannot be determined by static analysis alone.",
      "impact": "The routing logic for paragraph 0510-DISPATCH is incomplete in this BRD. One or more process paths may be undocumented.",
      "recommended_action": "Obtain runtime execution traces or interview original developers. Document all possible GO TO targets for the ALTER statement at line 612.",
      "source_program": "CBACT01C",
      "source_paragraph": "0500-ROUTE",
      "source_line": 612,
      "detected_in": ["parser_artifact", "logic_artifact"],
      "related_rule_ids": [],
      "related_gap_ids": ["GAP-007"]
    },
    {
      "gap_id": "GAP-002",
      "gap_type": "UNRESOLVED_CALL",
      "severity": "high",
      "title": "CALL to EXTPROG1 cannot be resolved",
      "description": "Program CBACT01C makes a static CALL to program EXTPROG1 at line 820, but EXTPROG1 is not present in the repository. It may be a vendor-supplied utility, an OS service, or a program in a separate load library.",
      "impact": "The behaviour triggered by this CALL is absent from the BRD. The process flow diagram for CBACT01C shows this as an unresolved step.",
      "recommended_action": "Determine if EXTPROG1 is in a load library, vendor package, or OS utility. Document its interface and expected behaviour and add to the BRD manually.",
      "source_program": "CBACT01C",
      "source_paragraph": "0800-EXTERNAL-CALL",
      "source_line": 820,
      "detected_in": ["inventory_artifact"],
      "related_rule_ids": [],
      "related_gap_ids": []
    },
    {
      "gap_id": "GAP-012",
      "gap_type": "SME_REVIEW_REQUIRED",
      "severity": "high",
      "title": "Interest calculation rate source unconfirmed — BR-098",
      "description": "Business rule BR-098 (Calculate interest on outstanding balance) uses field WS-INT-RATE in its calculation, but the source of this rate value cannot be determined statically. It may be loaded from a parameter file, a DB2 table, or a hardcoded constant in an external copybook.",
      "impact": "The interest calculation rule may be documented with an incorrect or incomplete rate source. The BRD flags this with a medium confidence rating.",
      "recommended_action": "Review the full execution trace for program CBINT01C and confirm where WS-INT-RATE is set before paragraph 1000-CALC-INTEREST is called.",
      "source_program": "CBINT01C",
      "source_paragraph": "1000-CALC-INTEREST",
      "source_line": 342,
      "detected_in": ["rules_artifact"],
      "related_rule_ids": ["BR-098"],
      "related_gap_ids": []
    }
  ],
  "assumptions": [
    {
      "assumption_id": "ASM-001",
      "description": "Programs with subtype 'batch' are assumed to run in a z/OS JES2 batch environment unless their JCL indicates otherwise.",
      "basis": "inventory_artifact program subtype classification",
      "confidence": "high"
    },
    {
      "assumption_id": "ASM-002",
      "description": "REDEFINES relationships with matching byte lengths are assumed to be intentional — they represent the same storage area interpreted in two different ways.",
      "basis": "data_artifact redefines_groups",
      "confidence": "high"
    },
    {
      "assumption_id": "ASM-003",
      "description": "88-level condition names are assumed to represent active business rules, not legacy or retired conditions.",
      "basis": "No tombstone comments or version flags detected in condition definitions",
      "confidence": "medium"
    }
  ]
}
```

---

## gaps_register.md format

Write the human-readable register as a Markdown document with this structure:

```markdown
# Gaps and assumptions register

**System:** {SYSTEM_NAME}
**Generated:** {timestamp}
**Total gaps:** {N} ({critical} critical, {high} high, {medium} medium, {low} low)

---

## Critical gaps — must resolve before BRD sign-off

### GAP-001 — ALTER statement in CBACT01C
**Type:** ALTER statement | **Program:** CBACT01C | **Line:** 612

{description}

**Impact:** {impact}

**Recommended action:** {recommended_action}

---

## High severity gaps

...

## Medium severity gaps

...

## Low severity gaps

...

## Assumptions

| ID | Assumption | Basis | Confidence |
|---|---|---|---|
| ASM-001 | ... | ... | High |

```

---

## Error handling

| Condition | Action |
|---|---|
| Artifact file not found | Log `error`, skip that artifact's gap sources, note in register |
| Gap deduplication produces conflicting severities | Use highest severity, note both in `detected_in` |
| No gaps found | Write register with zero gaps and note pipeline completed cleanly |
| More than 100 gaps detected | Group by program and type, write summary table first then detail |
