
# Cortex OS — Agent Instructions (Contracts Repo)

This repository is the **canonical source of truth** for:
- API schemas (OpenAPI / JSON)
- IPC command contracts (Wiring Matrix)
- Conventions (Errors, Pagination, Naming)

---

## Documentation Hierarchy

```text
docs/
├── VERSIONING.md                         # Semver + breaking change policy
├── CONVENTIONS.md                        # Naming, errors, IDs, pagination, dates
├── CODEGEN.md                            # tauri-specta generation flow
└── technical_planning/
    └── 002_IPC_WIRING_MATRIX.md          # [Source of Truth] IPC commands
```

> **Note:** The wiring matrix currently uses Phase 0 bridge commands (dot-notation, domain-specific). The target-state commands use snake_case page-centric naming per `001_architecture.md` Section 6.2 and `CONVENTIONS.md`. See the reconciliation note at the top of `002_IPC_WIRING_MATRIX.md`.

---

## Change Rules
1. **Backward Compatibility:** All changes must be evaluated for breaking impact. See `docs/VERSIONING.md`.
2. **Paired PRs:**
   - FE repo must update generated types.
   - BE repo must update handlers.
3. **Wiring Matrix:** Every IPC command must have a side-effect description.
4. **ADR Compliance:** Schema changes must align with ADR-0006 (EAV/Page model). IPC naming must follow `docs/CONVENTIONS.md`.
5. **Frontend Consumption Discipline:** Any FE follow-up work from contract changes must follow ADR-0017 hooks controller rules (`hooks/use*.ts`, no direct `services/` imports in views, hook tests in `tests/hooks/`).

---

## Before You Start
1. Read `docs/VERSIONING.md` for breaking change policy.
2. Read `docs/CONVENTIONS.md` for naming and type conventions.
3. Update `002_IPC_WIRING_MATRIX.md` first before implementing in BE/FE.
