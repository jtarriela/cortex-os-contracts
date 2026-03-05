# Contracts Versioning Policy

**Status:** Draft v1
**Date:** 2026-02-19

---

## 1) Version Scheme

The contracts package uses [Semantic Versioning 2.0](https://semver.org/):

```
MAJOR.MINOR.PATCH
```

| Component | Increment When |
|-----------|---------------|
| MAJOR | Breaking changes to existing commands (removal, renamed fields, changed types) |
| MINOR | New commands added, new optional fields added to existing commands |
| PATCH | Documentation fixes, typo corrections, no contract changes |

**Current version:** `0.10.10` (pre-release, all changes are considered non-breaking until 1.0)

---

## 2) Breaking vs. Non-Breaking Changes

### Breaking Changes (require MAJOR bump after 1.0)

- Removing a command
- Renaming a command
- Removing a field from a response type
- Changing a field's type (e.g., `string` → `number`)
- Making an optional request field required
- Changing the semantics of an existing field
- Renaming a field

### Non-Breaking Changes (MINOR bump)

- Adding a new command
- Adding a new optional field to a request type
- Adding a new field to a response type (clients must tolerate unknown fields)
- Adding a new error code
- Adding new enum values to a `select` type (clients must tolerate unknown values)

Example (ADR-0018 E23): adding `calendar_get_range` / `calendar.getRange` is a **MINOR** contract change.
Example (ADR-0019 V1): adding `obsidian_*` command surface and `AISettings.embeddingProvider` is a **MINOR** contract change.

### No Version Change (PATCH)

- Documentation updates
- Adding or updating examples
- Internal restructuring that doesn't change the contract surface

---

## 3) Deprecation Policy

Deprecated commands and fields remain functional for **2 minor versions** after deprecation is announced.

### Deprecation Process

1. **Announce:** Add a `@deprecated` annotation in the contract definition and a note in the wiring matrix
2. **Warn:** The backend logs a warning when a deprecated command is invoked
3. **Remove:** After 2 minor versions, the command/field is removed (MAJOR bump if post-1.0)

### Example Timeline

```
v0.3.0 — `tasks.create` deprecated (use `vault_create_page`)
v0.4.0 — `tasks.create` still works, emits deprecation warning
v0.5.0 — `tasks.create` removed
```

---

## 4) Pre-1.0 Policy

While the version is `0.x.y`:

- MINOR bumps may include breaking changes (standard semver pre-release behavior)
- Breaking changes should still be documented in the changelog
- The deprecation policy is relaxed — deprecated commands may be removed in the next minor version
- This allows rapid iteration during the Phase 0 → Phase 1 transition

---

## 5) Cross-Repo Coordination

Per `CONTRIBUTING.md` and `.system/PROTOCOL.md`:

1. Contract changes land in `cortex-os-contracts` first
2. Paired PRs in `cortex-os-frontend` and `cortex-os-backend` reference the contracts PR
3. The contracts version is bumped in the same PR as the contract change
4. Frontend and backend `package.json` / `Cargo.toml` reference the contracts version

---

## 6) Changelog

All contract changes are documented in `CHANGELOG.md`. Each entry includes:

- Version number
- Date
- List of added/changed/deprecated/removed commands and fields
- Reference to the relevant ADR or issue

---

## 7) Calendar Compatibility Governance (ADR-0018 E27)

Calendar contract changes carry elevated regression risk because DayFlow adapter behavior, backend sync semantics, and frontend interaction guards all depend on stable payload fields.

### Changes that require calendar compatibility review

- any add/remove/rename/type-change for calendar request/response fields
- calendar mutation error-shape changes (`INVALID_INPUT` semantics, message behavior)
- command behavior changes that alter read-only/editability policy interpretation
- date/time boundary or interval validation rule changes

### Mandatory paired validation before merge

For calendar-surface contract changes, contracts PRs must link:

- paired backend implementation PR with passing storage/app tests
- paired frontend implementation PR with passing adapter + guardrail tests
- evidence that wiring matrix, versioning, and changelog entries are synchronized

### Reviewer checklist snippet (calendar change gate)

```text
- [ ] Calendar compatibility impact assessed (breaking/non-breaking classification)
- [ ] Paired backend PR linked with test evidence (`cargo test -p cortex-storage`, `cargo test -p cortex-app`)
- [ ] Paired frontend PR linked with test evidence (`npm run test:dayflow-guardrails`)
- [ ] `002_IPC_WIRING_MATRIX.md` calendar section updated
- [ ] Changelog/version bump policy applied per SemVer rules
```

### Google Mirror / Full-Sync Compatibility Notes (2026-02-22)

The following contract additions are classified as **MINOR** changes (additive fields/commands):

- `integrations.googleCalendars` response expanded from `string[]` to metadata objects (`id`, `summary`, `backgroundColor?`, `primary`)
- `IntegrationSettings` additive fields: `editableCalendars`, `mirrorMigrationV1Done`
- new command: `integrations.deleteMirroredEvent`
- new event stream payload: `integrations_sync_progress`

Frontend and backend must treat unknown response/event fields as forward-compatible.
