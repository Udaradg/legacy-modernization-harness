# legacy-modernization-harness

This repository is a harness for legacy modernization using AI agents (Anthropic Claude and GitHub Copilot).
The basic idea is to orchestrate specialized agents to analyze, model, and synthesize documentation for COBOL-based systems so teams can modernize safely and systematically.

Pipeline phases

Phase 1 — Ingestion & Inventory: An Inventory Agent scans the COBOL codebase, catalogs all programs, copybooks, JCL, and data files, and builds a dependency graph.

Phase 2 — Structural Parsing: A Parser Agent extracts raw structure — DIVISION/SECTION layouts, paragraph names, WORKING-STORAGE entries, FD/SD definitions, and PERFORM/CALL chains — without trying to interpret meaning yet.

Phase 3 — Data Modeling: A Data Agent takes all COPY, FD, 01-level records, and Working Storage entries and produces a normalized data dictionary, ER relationships, and a canonical data model.

Phase 4 — Logic Extraction: A Logic Agent reads each paragraph and section, traces control flow (EVALUATE/PERFORM/GOTO), and produces pseudocode + annotated flowcharts per program.

Phase 5 — Business Rule Mining: A Rules Agent takes the pseudocode and flags domain-significant conditions (thresholds, validation rules, branching logic) and maps them to named business rules.

Phase 6 — Component Diagramming: A Diagram Agent takes the call/PERFORM graph and produces component, sequence, and process-flow diagrams in structured format.

Phase 7 — BRD Synthesis: A Synthesis Agent assembles everything — inventory, data model, rules, diagrams, process flows — into a structured Business Requirements Document.

## Generated Artifacts (`output/`)

All artifacts below were produced by GitHub Copilot agents running the pipeline phases above against a sample IBM z/OS COBOL codebase (Portfolio Management System).

### `output/inventory/`
| File | Description |
|------|-------------|
| `inventory_artifact.json` | Full program inventory: 83 source files scanned — 28 programs (11 batch, 8 online, 6 common, 7 portfolio, 2 test, 3 utility), 24 copybooks, categorized with metrics |

### `output/parser/`
| File | Description |
|------|-------------|
| `parser_artifact.json` | Parsing summary: 39 programs parsed, 404 paragraphs, 444 PERFORM edges, 0 GOTO edges |
| `raw_structure/*.json` (39 files) | Per-program parse trees: DIVISION/SECTION layouts, paragraph names, FD/SD definitions, WORKING-STORAGE entries, and PERFORM/CALL chains |

### `output/data/`
| File | Description |
|------|-------------|
| `data_artifact.json` | Data modeling summary: record counts, field counts, and condition tallies across the full data dictionary |
| `data_layouts/*.json` (57 files) | Per-copybook/per-working-storage data layouts: field names, PIC clauses, parsed formats, byte offsets, hierarchical structure, and which programs use each record |

### `output/logic/`
| File | Description |
|------|-------------|
| `logic_artifact.json` | Logic extraction summary: 38 programs processed, 0 failures |
| `program_logic/*.json` (38 files) | Per-program execution traces: entry points, step-by-step paragraph flows, conditions, line numbers, and control-flow annotations |

### `output/rules/`
| File | Description |
|------|-------------|
| `rules_artifact.json` | Rules summary: 90 total rules extracted, 11 high-confidence, 17 flagged as duplicates |
| `classified_conditions.json` | 192 classified conditions categorized by pattern (e.g., `ERROR_HANDLING`, `FIELD_VALUE_COMPARE`) with source references |

### `output/diagram/`
| File | Description |
|------|-------------|
| `diagrams_artifact.json` | Diagram generation metadata |
| `component_overview.mmd` + `component_*.mmd` (11 files) | Mermaid component diagrams showing program-level architecture and inter-program call relationships |
| `flow_*.mmd` (23 files) | Mermaid flowcharts showing intra-program control flow per program |
| `sequence_*.mmd` (5 files) | Mermaid sequence diagrams for key interaction flows |
| `erd.mmd` | Mermaid entity-relationship diagram for the full canonical data model |

### `output/final_report/`
| File | Description |
|------|-------------|
| `brd.md` | Full reverse-engineered Business Requirements Document (9+ chapters: system overview, business areas, functional requirements, data dictionary, process flows, business rules, gap analysis) |
| `brd_summary.md` | Executive summary of the BRD focused on portfolio lifecycle, transaction processing, online inquiry, and batch reporting |
| `gaps_register.json` | Structured gap register: 14 total gaps — 0 critical, 2 high, 5 medium, 7 low — categorized by type (e.g., `UNRESOLVED_CALL`, `UNRESOLVED_DATA_REFERENCE`) |
| `gaps_register.md` | Human-readable version of the gaps and assumptions register |


