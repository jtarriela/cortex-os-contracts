# Code Generation

**Status:** Draft v1
**Date:** 2026-02-19
**Scope:** How IPC contracts are defined in Rust and consumed as TypeScript types in the frontend

---

## 1) Overview

Cortex OS uses **tauri-specta** to auto-generate TypeScript type bindings from Rust struct and command definitions. This eliminates hand-maintained type duplication between the backend and frontend. The contracts repo documents the command surface; the actual type definitions live in Rust code and are generated into TypeScript.

```
┌──────────────────────┐     tauri-specta     ┌──────────────────────┐
│  Rust Backend        │ ──────────────────>  │  Frontend            │
│  #[tauri::command]   │                      │  services/backend.ts │
│  Rust structs        │                      │  (generated types)   │
└──────────────────────┘                      └──────────────────────┘
```

---

## 2) How It Works

### Defining Commands (Backend)

Commands are Rust functions annotated with `#[tauri::command]` and registered with `specta`:

```rust
#[tauri::command]
#[specta::specta]
async fn vault_create_page(
    kind: String,
    props: HashMap<String, serde_json::Value>,
    body: Option<String>,
) -> Result<PageRef, AppError> {
    // ...
}
```

Structs used in command signatures must derive `serde::Serialize`, `serde::Deserialize`, and `specta::Type`:

```rust
#[derive(Serialize, Deserialize, specta::Type)]
pub struct PageRef {
    pub page_id: String,
    pub path: String,
    pub title: String,
}
```

### Generating TypeScript (Build Step)

During the build, `tauri-specta` generates a TypeScript file with typed `invoke()` wrappers:

```typescript
// Auto-generated — do not edit
export async function vault_create_page(
    kind: string,
    props: Record<string, any>,
    body?: string
): Promise<PageRef> {
    return invoke("vault_create_page", { kind, props, body });
}

export interface PageRef {
    page_id: string;
    path: string;
    title: string;
}
```

### Output Location

Current checked-in generation output for the performance refactor tranche is written to:

```
frontend/services/generated/phase03-bindings.ts
```

Source contract spec:

```
contracts/specs/phase03-view-bindings.json
```

Generator entrypoint:

```
node contracts/scripts/generate-phase03-bindings.mjs
```

The long-term target remains a full generated backend client surface, but the current phase-0/3 slice already uses a generated module for the hot-surface command set instead of hand-maintaining those bindings inside `frontend/services/backend.ts`.

---

## 3) Generation Triggers

| Trigger | When |
|---------|------|
| `cargo tauri dev` | On backend code changes during development |
| `cargo tauri build` | Before production build |
| Manual | `node contracts/scripts/generate-phase03-bindings.mjs` (current phase-0/3 binding slice) |

---

## 4) Manual Extensions

Sometimes the generated types need augmentation:

- **Computed fields** (e.g., `durationMinutes` derived from `start` and `end`) — add to a `frontend/src/services/backend.extensions.ts` file that re-exports generated types with added fields
- **Frontend-only types** (e.g., UI state, animation flags) — define in `frontend/src/types.ts`, never in the generated file

**Rule:** Never edit generated binding files directly. Update the contract source/spec and regenerate instead.

---

## 5) CI Verification

The CI pipeline verifies that generated types are up-to-date:

```bash
# Generate fresh bindings for the current phase-0/3 command slice
node contracts/scripts/generate-phase03-bindings.mjs

# Check for uncommitted changes
git diff --exit-code frontend/services/generated/phase03-bindings.ts
```

If the generated file differs from what's committed, the CI check fails. This ensures that:

1. Rust type changes are always reflected in the TypeScript bindings
2. No manual edits have been made to the generated file
3. The frontend and backend are always in sync

---

## 6) Type Mapping

| Rust Type | TypeScript Type |
|-----------|----------------|
| `String` | `string` |
| `i32`, `i64`, `u32`, `u64` | `number` |
| `f32`, `f64` | `number` |
| `bool` | `boolean` |
| `Option<T>` | `T \| null` |
| `Vec<T>` | `T[]` |
| `HashMap<String, T>` | `Record<string, T>` |
| `serde_json::Value` | `any` (prefer specific types) |
| Custom struct | Generated interface |
| Enum (unit variants) | String union (`"A" \| "B" \| "C"`) |
| Enum (data variants) | Tagged union |
| `Result<T, E>` | Resolved to `T` (errors thrown as exceptions) |

---

## 7) Transition from dataService.ts

During the current performance refactor tranche:

1. `phase03-bindings.ts` is generated from `contracts/specs/phase03-view-bindings.json`
2. `frontend/services/backend.ts` consumes that generated module for the new hot-surface commands
3. Existing hand-maintained wrappers remain only for legacy/non-migrated command surfaces
4. Future phases can widen generation scope until the whole backend client is generated

The test strategy (ADR-0012) ensures that tests written against `dataService.ts` contracts transfer seamlessly — the test assertions remain valid, only the import path changes.

---

## 8) Calendar Contract Change Workflow (ADR-0018 E27)

When calendar command signatures or payload fields change, generation alone is not sufficient. Use this workflow to keep frontend/backend consumers compatible.

### Required flow

1. Update `docs/technical_planning/002_IPC_WIRING_MATRIX.md` calendar section first.
2. Regenerate/align consumer bindings and callsites in frontend/backend.
3. Run consumer validation:
   - frontend: `npm run test:dayflow-guardrails`
   - backend: `cargo test -p cortex-storage` and `cargo test -p cortex-app`
4. Confirm versioning + changelog updates in this repo (`VERSIONING.md`, `CHANGELOG.md`).
5. Link paired PRs in contracts/frontend/backend before merge.

### Calendar change pre-merge checklist

```text
- [ ] Wiring matrix calendar entries updated and reviewed
- [ ] Frontend DayFlow adapter/guardrail suite passes
- [ ] Backend calendar storage/app suites pass
- [ ] Paired PR links included in all three repos
- [ ] Versioning/changelog updates are present and consistent
```
