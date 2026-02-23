
# MEMORY — Cortex OS (Contracts)

## Current Focus
- ADR-0019/ADR-0022 follow-on contract sync: Vault Workbench inspector IPCs (`search.graphLinks`, `obsidian.noteInspect`) and linked-note diagnostics payloads documented.

## System State (Facts Only)
- Canonical source of truth for API/IPC.
- Submodule in cortex-os integration.

## Active ADRs
- None yet.

## Recent Progress (Append-only)
- 2026-02-19: Documentation initialization.
- 2026-02-19: Phase 3 foundation contract update: documented realtime event streams (`page_created`, `page_updated`, `page_deleted`) and payload shape in `docs/technical_planning/002_IPC_WIRING_MATRIX.md`. Updated `CHANGELOG.md`.
- 2026-02-19: Phase 3 completion contract sync. Updated wiring matrix with payload corrections (`travel.createTrip`, `travel.createCard`, `habits.toggle`, `finance.importCsv`), added search semantic/graph contracts, and added secondary analytics contracts (`journal.query`, `journal.moodTrends`, `habits.getSummary`, `goals.getProgressSummary`, `finance.getSummary`, `travel.getItinerary`, `meals.getNutritionSummary`). CHANGELOG updated accordingly.
- 2026-02-21: ADR-0018 E27 governance updates for issue #17. Added calendar compatibility review checklist and paired-validation requirements in `docs/VERSIONING.md`, added calendar contract change workflow + pre-merge checklist in `docs/CODEGEN.md`, and added E27 governance note in calendar section of `docs/technical_planning/002_IPC_WIRING_MATRIX.md` referencing required frontend/backend validation evidence.
- 2026-02-23: Added Vault Workbench inspector contract surface for exact backlinks/outgoing links (`search.graphLinks`) and linked-note metadata inspection (`obsidian.noteInspect`). Updated IPC wiring matrix and bumped `CHANGELOG.md` with `0.9.0` entry.
