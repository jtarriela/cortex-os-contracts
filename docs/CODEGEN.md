# Code Generation

**Status:** Draft v1
**Date:** 2026-02-19
**Scope:** How IPC contracts are defined in Rust and consumed as TypeScript types in the frontend

---

## 1) Overview

Cortex OS plans to use **tauri-specta** to auto-generate TypeScript type bindings from Rust struct and command definitions. This eliminates hand-maintained type duplication between the backend and frontend. The contracts repo documents the command surface; the actual type definitions live in Rust code.

```
┌──────────────────────┐     tauri-specta     ┌──────────────────────┐
│  Rust Backend        │ ──────────────────>  │  Frontend            │
│  #[tauri::command]   │                      │  raw generated       │
│  Rust structs        │                      │  bindings (future)   │
└──────────────────────┘                      └──────────────────────┘
```

Current repo state:
- `frontend/services/backend.ts` is a hand-maintained facade.
- It preserves stable frontend-friendly function names, performs DTO normalization, and wraps raw `invoke()` calls.
- Any future generated bindings should feed that facade rather than overwrite it.

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

When enabled, `tauri-specta` can generate a TypeScript file with typed `invoke()` wrappers:

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

Generated TypeScript should be written to a dedicated raw-bindings file and imported by the hand-maintained facade. It must not overwrite:

```
frontend/services/backend.ts
```

Recommended pattern:
- generated raw bindings live in a separate file/path owned by codegen
- `frontend/services/backend.ts` wraps those bindings and preserves normalization plus compatibility behavior
- view components continue importing the facade, not the raw generated layer

---

## 3) Generation Triggers

| Trigger | When |
|---------|------|
| `cargo tauri dev` | On backend code changes during development |
| `cargo tauri build` | Before production build |
| Manual | `cargo test -p cortex_core -- generate_bindings` (for CI) |

---

## 4) Manual Extensions

Sometimes generated types need augmentation:

- **Computed fields** (e.g., `durationMinutes` derived from `start` and `end`) — add them in the hand-maintained bridge or companion extension files
- **Frontend-only types** (e.g., UI state, animation flags) — define in `frontend/types.ts`, never in the generated file

**Rule:** Never point codegen at `frontend/services/backend.ts`. That file is intentionally hand-maintained.

---

## 5) CI Verification

If codegen is enabled, CI should verify that the generated raw-bindings artifact is up-to-date:

```bash
# Generate fresh bindings
cargo test -p cortex_core -- generate_bindings

# Check for uncommitted changes in the generated artifact
git diff --exit-code <generated-bindings-path>
```

If the generated file differs from what's committed, the CI check fails. This ensures that:

1. Rust type changes are always reflected in the TypeScript bindings
2. No manual edits have been made to the generated artifact
3. The frontend raw bindings and backend are in sync

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

## 7) Current Frontend Boundary

Current state:

1. `frontend/services/backend.ts` is the active frontend data-access facade.
2. It is hand-maintained and contains normalization, compatibility shims, and domain-oriented helper names.
3. View components and hooks import that facade directly.
4. Future codegen should emit raw bindings underneath that facade, not replace it.

This keeps the contracts/backend surface explicit while preserving a stable frontend API boundary.

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
