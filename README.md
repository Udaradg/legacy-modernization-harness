# legacy-modernization-harness

This repository is a harness for legacy modernization using AI agents (Anthropic Claude and GitHub Copilot).
The basic idea is to orchestrate specialized agents to analyze, model, and synthesize documentation for COBOL-based systems so teams can modernize safely and systematically.

## Pipeline Phases

| Phase | Agent | What it does |
|-------|-------|-------------|
| 1 — Ingestion & Inventory | Inventory Agent | Scans the COBOL codebase, catalogs all programs, copybooks, JCL, and data files, and builds a dependency graph |
| 2 — Structural Parsing | Parser Agent | Extracts raw structure — DIVISION/SECTION layouts, paragraph names, WORKING-STORAGE entries, FD/SD definitions, and PERFORM/CALL chains — without trying to interpret meaning yet |
| 3 — Data Modeling | Data Agent | Takes all COPY, FD, 01-level records, and Working Storage entries and produces a normalized data dictionary, ER relationships, and a canonical data model |
| 4 — Logic Extraction | Logic Agent | Reads each paragraph and section, traces control flow (EVALUATE/PERFORM/GOTO), and produces pseudocode + annotated flowcharts per program |
| 5 — Business Rule Mining | Rules Agent | Takes the pseudocode and flags domain-significant conditions (thresholds, validation rules, branching logic) and maps them to named business rules |
| 6 — Component Diagramming | Diagram Agent | Takes the call/PERFORM graph and produces component, sequence, and process-flow diagrams in structured format |
| 7 — BRD Synthesis | Synthesis Agent | Assembles everything — inventory, data model, rules, diagrams, process flows — into a structured Business Requirements Document |

## Agent & Skill Breakdown

### 1. Inventory Agent
| Skill | Description |
|-------|-------------|
| **File Walker** | Recursively lists all `.cob`, `.cbl`, `.cpy`, `.jcl`, `.ctl` files; extracts program IDs and `COPY` statements |
| **Dependency Graph** | Builds a directed graph of `CALL`/`COPY`/`EXEC` relationships using a tool like NetworkX or a simple adjacency list in JSON |

### 2. Parser Agent
| Skill | Description |
|-------|-------------|
| **COBOL AST Parser** | Uses a COBOL grammar parser (e.g. `cobol-parser` npm package or GnuCOBOL with `-fsyntax-only`) to extract the raw AST; alternatively, Claude reads raw source and extracts structure in passes |
| **Section Mapper** | Maps each `DIVISION` → `SECTION` → `Paragraph` into a structured JSON schema |

### 3. Data Agent
| Skill | Description |
|-------|-------------|
| **Record Layout Parser** | Extracts all `01`-level group items, their children (`PIC` clauses, `REDEFINES`, `OCCURS`), and `COPY`-sourced layouts into a field-level dictionary |
| **Data Dictionary Builder** | Normalizes fields across all programs, resolves `REDEFINES` aliases, and identifies shared data structures |

### 4. Logic Agent
| Skill | Description |
|-------|-------------|
| **Control Flow Tracer** | Builds a call graph of `PERFORM`/`GO TO` chains per program; flags dead code, recursive performs, and exit conditions |
| **Pseudocode Generator** | Translates COBOL paragraphs to readable pseudocode with inline comments explaining COBOL-specific idioms (`88`-levels, `COMPUTE` rounding, `STRING`/`UNSTRING`) |

### 5. Rules Agent
| Skill | Description |
|-------|-------------|
| **Condition Classifier** | Reads `EVALUATE`/`IF`/`88`-level conditions and classifies them as validation rules, calculation rules, routing rules, or error-handling rules |
| **Rule Tagger** | Assigns human-readable names to each rule (e.g. `VALIDATE-CREDIT-LIMIT`, `CALC-LATE-PENALTY`) and maps each to the source paragraph |

### 6. Diagram Agent
| Skill | Description |
|-------|-------------|
| **Component Mapper** | Takes the call graph and produces Mermaid or PlantUML component diagrams |
| **Sequence Builder** | Produces sequence diagrams for key process flows (e.g. transaction processing, batch job runs) |

### 7. Synthesis Agent
| Skill | Description |
|-------|-------------|
| **Section Assembler** | Merges all prior artifacts into a structured BRD template (Executive Summary, Scope, Data Model, Process Flows, Business Rules catalog, Component Diagrams, Glossary) |
| **Gap Detector** | Identifies areas where logic was ambiguous, incomplete, or inferred — flags them as "requires SME review" |

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


