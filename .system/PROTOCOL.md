# Cortex — Common Development Protocol (.system/PROTOCOL.md)

> This file is **canonical** and should be kept **identical** across all Cortex repos:
>
> - `cortex-os` (integration/workspace)
> - `cortex-os-frontend`
> - `cortex-os-backend`
> - `cortex-os-contracts`
>
> Repo-specific doc paths and checklists are defined in **`AGENTS.md`** and `.system/DOC_SYNC_CHECKLIST.md` within each repo.

---

## 0) Purpose

This protocol enforces **high-signal, low-drift delivery** across a multi-repo system by combining:

- **Interface-First design**
- **TDD (Red → Green → Refactor)**
- **Atomic work units**
- **Doc-Sync merge gating**
- **Cross-repo contract governance**
- **ADR lifecycle discipline**

If a behavior exists in code, it must be discoverable in docs and traceable to tests.

---

## 1) Core Principles (Non-Negotiable)

1. **TDD First**
   - No feature implementation logic may be written until a failing test exists.
2. **Interface First**
   - Public surface area (types, signatures, messages, schemas) must be defined **before** tests.
3. **Atomic Tasks**
   - Break work into “atoms” with ≤ ~50 lines of net-new logic (excluding tests/docs).
4. **No Drift**
   - Canonical artifacts live in one place (see Ownership model below). Other repos consume via generation or submodule pinning.
5. **Doc-Sync Merge Gate**
   - A PR may not merge unless doc sync requirements are met per the repo’s `.system/DOC_SYNC_CHECKLIST.md`.
6. **ADR Lifecycle Sync**
   - When implementation starts or completes for ADR-scoped work, the ADR status must be updated in the same PR set (or paired PR set).
7. **Issue-Driven Branch Discipline**
   - If there is a GitHub issue, branch creation and linkage are mandatory.

---

## 2) Ownership Model (Canonical Source of Truth)

**Never maintain two hand-edited sources of truth for the same thing.**

### Repo ownership

- **`cortex-os` (integration)**: system-level docs, global FRs, global traceability, local-dev orchestration, submodule pinning
- **`cortex-os-frontend`**: frontend implementation + FE architecture docs (components, stores, routing, theming)
- **`cortex-os-backend`**: backend implementation + BE architecture docs (services, DB schema/migrations, internal APIs)
- **`cortex-os-contracts`**: canonical API/IPC contracts + versioning/conventions + wiring matrix

### “Derived artifacts” rule

Generated files (clients, stubs, types) are **derived** from `cortex-os-contracts` and must not be edited manually.

---

## 3) Role-Based Workflow (Default Execution Loop)

See `.system/ROLES.md` for full detail. The default loop is:

1) **Architect** → defines interfaces/contracts and constraints (writes ADR if needed)  
2) **Test-Gen** → writes failing tests aligned to interfaces and acceptance criteria  
3) **Coder** → implements minimal code to pass tests  
4) **Reviewer** → correctness/security/perf/design compliance  
5) **Doc-Sync** → updates MEMORY + docs + traceability + ADR status

**Hard rule:** If you are not in the Architect role, you do not invent new public interfaces.

---

## 4) Definitions

### Atom

A small change unit with:

- one clear objective
- a testable definition of done
- minimal logic footprint (≤ ~50 lines net-new logic)
- doc updates included

### Contract surface

Anything that crosses boundaries:

- IPC commands (`#[tauri::command]`, FE invoke payloads)
- API schemas (OpenAPI/proto/JSON schema)
- error envelopes, pagination, auth semantics
- persistent storage schema (DB tables/migrations)

---

## 5) Cross-Repo Change Policy (Paired PR Requirement)

If your change affects a contract surface owned by another repo, you **must** open a **paired PR** and link them.

### Mandatory paired PR triggers

1) **Backend adds/modifies `#[tauri::command]`**
   - Update wiring matrix in `cortex-os-contracts`
   - Update FE IPC client usage (if applicable) in `cortex-os-frontend`

2) **Contracts change (schemas / conventions / wiring matrix)**
   - Update FE generated client/types or usage in `cortex-os-frontend`
   - Update BE handler/validators in `cortex-os-backend`

3) **DB schema change**
   - Update BE schema docs in `cortex-os-backend`
   - If it affects external behavior or data contracts, update relevant contract docs

4) **System-level FR / traceability change**
   - Update in `cortex-os` (canonical)
   - Link implementing PRs in FE/BE/Contracts

### Integration pinning rule

After component PRs merge, the integration repo (`cortex-os`) must bump submodule SHAs in a follow-up “pinning PR” (or release PR) and record it in integration docs/MEMORY.

---

## 6) [STRICT] Delivery Protocol

### A) TDD & Verification

1. **Red first:** commit failing test(s) before implementation logic
2. **Green:** implement minimal code to pass
3. **Refactor:** clean up while keeping tests green
4. **Deterministic tests:** no flaky time/network randomness
5. **No test weakening:** do not “fix” failures by relaxing assertions unless Architect approves

### B) Interface Integrity (IPC / API)

1. **Do not invent interfaces** outside Architect role
2. **IPC canonical truth:** wiring matrix + message shapes are canonical in `cortex-os-contracts`
3. **State attribution:** if an IPC command changes frontend state, its side effects must be captured in the wiring matrix
4. **Error envelope discipline:** errors must be consistent with contract conventions (don’t ad-hoc new error shapes)

### C) Frontend Architecture Discipline (if working in FE repo)

1. **Single IPC Gateway:** all backend calls go through the central IPC client (per FE docs)
2. **No blind component creation:** check component catalog first
3. **No duplicate stores:** check store registry first
4. **Reactive invariants:** prefer store subscriptions over deep prop threading unless explicitly justified

### D) Backend Architecture Discipline (if working in BE repo)

1. **Service boundaries:** keep handlers thin; domain/services own logic
2. **Schema/migrations discipline:** any schema change must include migration strategy + docs update
3. **Input validation:** validate contract payloads at boundaries; fail fast with contract-compliant errors
4. **No silent breaking changes:** if an external behavior changes, update contracts + versioning

### E) Contracts Discipline (if working in Contracts repo)

1. **Contracts are code:** validate schemas, keep examples updated
2. **Breaking change policy:** must bump version per contracts versioning rules
3. **Changelog required:** record externally visible changes
4. **Paired PR enforcement:** require FE/BE implementing PRs for contract changes

### F) Traceability

1. Every shipped behavior must map:
   - **FR → repo → code → test evidence**
2. Update traceability artifacts per the repo’s `AGENTS.md` and doc sync checklist.

---

## 7) ADR Lifecycle Rules

### ADR states

- `PROPOSED` → decision drafted, not yet accepted for build
- `ACCEPTED` → chosen direction; implementation started or scheduled
- `IMPLEMENTED` → fully delivered with passing tests/evidence

### Transition rule (enforced)

- `PROPOSED` → `ACCEPTED` when implementation begins (issues/PRs in progress)
- `ACCEPTED` → `IMPLEMENTED` only when ADR-scoped issues are complete with test evidence

If ADRs are owned in a specific repo (system vs FE vs BE vs contracts), update the authoritative ADR file there.

---

## 8) GitHub Issue + Branch Protocol (Mandatory When Issue Exists)

1. Create branch **before** code changes:
   - `codex/issue-<id>-<slug>`
2. Post an issue comment:
   - “Branch `codex/issue-<id>-<slug>` created; work started.”
3. Commits and PR must reference the issue:
   - include `#<id>` in PR body and/or commits
4. Close issue only when acceptance criteria are met with passing tests + doc sync.

---

## 9) PR Body Minimum Standard (Template)

Include this checklist in the PR description (adapt per repo):

````text
- [ ] Tests added/updated (TDD) and passing
- [ ] Interfaces/contracts unchanged OR approved by Architect
- [ ] Doc-Sync checklist completed for this repo
- [ ] Paired PR(s) linked (if cross-repo contract/doc surface changed)
- [ ] ADR status updated (if ADR-scoped work started/completed)
- [ ] Traceability updated (where canonical)
````

------

## 10) Merge Gate Policy

A PR **must not merge** unless:

- Repo CI passes
- `.system/DOC_SYNC_CHECKLIST.md` requirements are met
- Any required paired PRs are linked and consistent
- ADR status transitions (if applicable) are correct

Branch protection must include the Doc-Sync guard status checks as required.

------

## 11) Escalation Rules (Stop-the-line)

Stop and escalate to the Architect (or open an ADR) if:

- implementation requires changing a public interface not previously defined
- you discover a mismatch between contracts and existing behavior
- a change introduces a breaking contract change without versioning agreement
- scope exceeds an atom (split into smaller issues)

------

## 12) Non-Goals

This protocol does **not** prescribe:

- exact folder structure (see each repo’s `AGENTS.md`)
- exact test frameworks (but requires deterministic TDD)
- exact CI implementation (but requires doc sync enforcement)

------

> If there is a conflict between this protocol and a repo’s `AGENTS.md`, treat `AGENTS.md` as the path authority (what/where), and this protocol as the process authority (how/when).
