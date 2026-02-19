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

Generated TypeScript is written to:

```
frontend/src/services/backend.ts
```

This file replaces the current `dataService.ts` as the frontend's data access layer. View components import from `backend.ts` instead of calling `dataService` functions.

---

## 3) Generation Triggers

| Trigger | When |
|---------|------|
| `cargo tauri dev` | On backend code changes during development |
| `cargo tauri build` | Before production build |
| Manual | `cargo test -p cortex_core -- generate_bindings` (for CI) |

---

## 4) Manual Extensions

Sometimes the generated types need augmentation:

- **Computed fields** (e.g., `durationMinutes` derived from `start` and `end`) — add to a `frontend/src/services/backend.extensions.ts` file that re-exports generated types with added fields
- **Frontend-only types** (e.g., UI state, animation flags) — define in `frontend/src/types.ts`, never in the generated file

**Rule:** Never edit `backend.ts` directly. It is regenerated on every build.

---

## 5) CI Verification

The CI pipeline verifies that generated types are up-to-date:

```bash
# Generate fresh bindings
cargo test -p cortex_core -- generate_bindings

# Check for uncommitted changes
git diff --exit-code frontend/src/services/backend.ts
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

During Phase 1 IPC wiring:

1. `backend.ts` is generated with typed invoke wrappers
2. View components are updated to import from `backend.ts` instead of `dataService.ts`
3. `dataService.ts` is gradually deprecated (functions removed as their `backend.ts` equivalents are verified)
4. Once all views use `backend.ts`, `dataService.ts` is deleted

The test strategy (ADR-0012) ensures that tests written against `dataService.ts` contracts transfer seamlessly — the test assertions remain valid, only the import path changes.
