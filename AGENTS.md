
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
├── CONVENTIONS.md                        # Naming, errors, IDs
├── CODEGEN.md                            # How FE/BE generate stubs
└── technical_planning/
    └── 002_IPC_WIRING_MATRIX.md          # [Source of Truth] IPC commands
```

---

## Change Rules
1. **Backward Compatibility:** All changes must be evaluated for breaking impact.
2. **Paired PRs:** 
   - FE repo must update generated types.
   - BE repo must update handlers.
3. **Wiring Matrix:** Every IPC command must have a side-effect description.

---

## Before You Start
1. Read `docs/VERSIONING.md`.
2. Update `002_IPC_WIRING_MATRIX.md` first before implementing in BE/FE.
