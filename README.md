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

See the `output/` folder for generated artifacts (inventory, parser outputs, data layouts, diagrams, rules, and final BRD).


