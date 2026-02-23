# Contracts Changelog

All notable changes to the IPC contract surface are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [0.10.1] — 2026-02-23

### Added
- Travel v2 Stage 1-2 structured planning contract surface:
  - `travel.getWorkspace`
  - `travel.createLocation`
  - `travel.updateLocation`
  - `travel.reorderLocations`
  - `travel.createItem`
  - `travel.updateItem`
  - `travel.moveItem`
  - `travel.reorderItems`
  - `travel.createExpense`
  - `travel.updateExpense`
  - `travel.deleteExpense`
  - `travel.getBudgetSummary`
  - `travel.legacyMigrateCards`

### Changed
- Travel contract docs now document the dual-read migration model for Travel v2:
  - legacy `travel.createCard` / `travel.getItinerary` remain supported as compatibility commands
  - Travel v2 workspace hydration uses `travel.getWorkspace` for structured `trip_location` / `trip_item` / `trip_expense` projections

## [0.10.0] — 2026-02-23

### Added
- Finance YNAB view-only integration contract surface:
  - `finance.ynabStatus`
  - `finance.ynabConnectPat`
  - `finance.ynabDisconnect`
  - `finance.ynabSync`
  - `finance.ynabGetTrackerConfig`
  - `finance.ynabSaveTrackerConfig`
  - `finance.ynabGetAnalytics`

### Changed
- Finance contract docs refined to reflect the Settings-driven finance UX split (Manual vs YNAB):
  - `finance.getBudget` is documented as the current manual `budget_month` page source used by the manual budget planner UI
  - `finance.ynabStatus` / `finance.ynabConnectPat` / `finance.ynabDisconnect` frontend usage is now documented under Settings → Integrations (with Finance using status as a gate/redirect)
- Finance contract docs distinguish manual finance commands from YNAB-backed local sync/analytics commands and document YNAB sync request fields (`budgetId`, `mode`, `writeCsv`, `reanalyze`).

## [0.9.0] — 2026-02-23

### Added
- Exact graph-link inspection contract for note inspector/backlinks UX:
  - `search.graphLinks` / `search_graph_links`
  - request shape: `{ pageId, direction?: "incoming" | "outgoing" | "both", limit? }`
  - response: `GraphLinkResult[]` (direction-tagged edges with relation/weight + source/target ids)
- Linked-note inspector metadata contract for Vault Workbench right drawer:
  - `obsidian.noteInspect` / `obsidian_note_inspect`
  - request shape: `{ page_id }`
  - response: `LinkedNoteInspectorStatus | null` (null for non-linked notes)

### Changed
- IPC wiring matrix now distinguishes exact directed link inspection (`search.graphLinks`) from traversal-based neighborhood search (`search.graphNeighbors`).
- Obsidian linked-vault contract section now documents note-level inspector metadata payloads (manifest/index/sync summaries) used by the Vault Workbench `Inspect` panel.

## [0.8.0] — 2026-02-22

### Added
- Linked-note write-back contract for Obsidian linked vaults:
  - `obsidian.noteSave` / `obsidian_note_save`
  - request shape: `{ page_id, base_hash, markdown }`
  - response union: `LinkedNoteSaveResult`
    - `saved`: `{ status: "saved", note, sourceHash }`
    - `conflict`: `{ status: "conflict", serverMarkdown, serverHash, message }`

### Changed
- `VaultLink.mode` contract expanded from `read_only` to `read_only | read_write`.
- `obsidian.linkAdd` and `obsidian.linkSetMode` now accept `mode: "read_only" | "read_write"`.
- Linked-vault conflict policy documented as status-return semantics (no ad-hoc error envelope changes).

## [0.7.0] — 2026-02-22

### Added
- ADR-0019 linked-vault contract surface:
  - `obsidian.linkAdd` / `obsidian_link_add`
  - `obsidian.linkList` / `obsidian_link_list`
  - `obsidian.linkRemove` / `obsidian_link_remove`
  - `obsidian.linkSetMode` / `obsidian_link_set_mode`
  - `obsidian.syncNow` / `obsidian_sync_now`
  - `obsidian.syncStatus` / `obsidian_sync_status`
- New event stream contract:
  - `obsidian_sync_progress` with phase/progress payload for linked-vault sync UI.
- Project milestone dual-write contract notes and CRUD mapping via canonical page commands:
  - `projects.milestones.list` => `collection_query("col_project_milestones")`
  - `projects.milestones.create` => `vault_create_page(kind:"project_milestone", props)`
  - `projects.milestones.update` => `page_update_props(page_id, props)`
  - `projects.milestones.delete` => `vault_delete(page_id)`
- Explicit milestone/body checklist sync policy:
  - marker format `<!-- milestone:{checklist_anchor_id} -->`
  - on-save synchronization
  - conflict policy `milestone-page-wins`
  - markerized checklist deletion => milestone page deletion
  - checkbox mapping `checked => COMPLETED`, `unchecked => NOT_STARTED`

### Changed
- Search/AI conventions now document ADR-0019 embedding provider policy:
  - `AISettings.embeddingProvider` = `same_as_model | openai | gemini | ollama | hash`
  - fallback from unsupported provider embeddings to deterministic `hash`.
- `tasks.update` request schema now documents timeline planning fields and dependency payload:
  - `planned_start_date`, `planned_end_date`
  - `baseline_start_date`, `baseline_end_date`
  - `actual_start_date`, `actual_end_date`
  - `dependencies[]` (`FS` + `lag_days?`)
- `projects.update` row now clarifies that milestones are handled via dual-write body + milestone pages rather than `props.milestones`.

## [0.6.1] — 2026-02-21

### Changed
- Added ADR-0018 E27 calendar compatibility governance:
  - `docs/VERSIONING.md` now includes mandatory calendar change review criteria and paired validation checklist.
  - `docs/CODEGEN.md` now includes calendar contract workflow steps requiring frontend/backend validation evidence before merge.
  - `docs/technical_planning/002_IPC_WIRING_MATRIX.md` calendar section now includes explicit E27 upgrade-governance note and required paired test evidence.

## [0.6.0] — 2026-02-20

### Added
- Calendar workspace range retrieval contract:
  - `calendar.getRange` / `calendar_get_range(startDate, endDate)`
  - request boundaries accept ISO datetime or `YYYY-MM-DD`
  - date-window semantics explicitly documented as inclusive `startDate`, exclusive `endDate`

### Changed
- Tauri argument casing notes now include `calendar_get_range` (`startDate`, `endDate`).
- Versioning policy example updated to classify this addition as a MINOR bump.

## [0.5.4] — 2026-02-20

### Changed
- Travel command contract now documents optional trip budget metadata:
  - `travel.createTrip` request includes `budget?` in addition to `title`, `destination`, `startDate`, and `endDate`.
  - Tauri argument-casing notes now include the optional `budget` key in the `travel_create_trip` invoke example.

## [0.5.3] — 2026-02-20

### Changed
- Clarified Tauri invoke argument casing in the IPC wiring matrix:
  - top-level command args are camelCase (`collectionId`, `pageId`, `startDate`, etc.)
  - nested `request` payload fields remain snake_case where documented.
- Updated matrix request-field examples for affected commands (`collection_query`, `calendar_get_week`, `travel_*`, `habits_toggle`, `journal_*`, `goals_get_progress_summary`, `meals_get_nutrition_summary`, `search_graph_*`, `review_*`, `secret_get/delete`) to reflect runtime-validated casing.

## [0.5.2] — 2026-02-20

### Changed
- IPC wiring matrix now explicitly documents canonical Phase 5 payload alignment:
  - `vault_create_page` canonical request is `{ kind, props, body? }`; `props.title` is canonical title source.
  - `capture.save` canonical request field is `text` (`content?` accepted as legacy compatibility).
- Matrix now documents markdown persistence side effects for:
  - `vault_create_page`
  - `page_update_body`
  - `save_commit`
  - `vault_delete`
  - `travel.createTrip`
  - `travel.createCard`
  - `capture.save`
- Integrations section now documents Google OAuth env prerequisites:
  - `GOOGLE_CLIENT_ID`
  - `GOOGLE_CLIENT_SECRET`

## [0.5.1] — 2026-02-20

### Changed
- Temporary voice-default adjustment while local Whisper runtime is deferred:
  - `AISettings.sttProvider` default is now `gemini` (online STT).
  - `local_whisper` remains in the enum and command contracts as a deferred selectable path.

## [0.5.0] — 2026-02-20

### Changed
- Voice IPC contracts now carry provider-aware semantics per ADR-0013:
  - `ai_transcribe` request accepts `tier?` and `mime_type?`, and routes via `AISettings.sttProvider` (`local_whisper|openai|gemini`).
  - `ai_synthesize` routes via `AISettings.ttsProvider` (`gemini|openai|local`) while preserving optional `voice`.
- Wiring matrix now documents STT/TTS provider selection as part of backend handler behavior.

## [0.4.0] — 2026-02-19

### Added
- Vault onboarding commands:
  - `vault_get_profile`
  - `vault_create`
  - `vault_select`
- Secret storage contract surface:
  - `secret_set`
  - `secret_get`
  - `secret_delete`
- Save-commit/indexing contracts:
  - `save_commit`
  - `index_queue_status`
- RAG command contracts:
  - `ai_rag_query`
  - `ai_suggest_links`
- AI stream error payload is now implemented and emitted by backend provider adapters (`ai_stream_error`).

### Changed
- `settings_get` / `settings_update` now use masked secret placeholders; plaintext keys are never returned over IPC.
- AI model capabilities (`text|image|audio|live`) are backend-driven and intended to gate frontend toggles.

### Migration Notes (FE/BE)
- Frontend should route note persistence through `save_commit` to get deterministic save-ACK indexing semantics.
- Frontend should treat `********` as a secret placeholder and avoid re-submitting unchanged key values.
- Backend should persist provider credentials via `secret_*` commands/storage and keep `ai_settings` key fields masked.

## [Unreleased] — Phase 0.5

### Added
- `calendar.getToday` — returns `CalendarEvent[]` for today's schedule (ADR-0007)
- `calendar.addEvent` — now covers TodayDashboard scheduling; `type` enum documented (`event|task|reminder|deep-work`)
- `meals.update` — update an existing meal record (FR-013)
- `meals.delete` — remove a meal from the weekly planner (FR-013)
- `recipes.update` — update recipe fields including `image_url` (FR-013)
- `recipes.delete` — delete a recipe (FR-013)
- Event stream contract section for `page_created`, `page_updated`, and `page_deleted` realtime invalidation payloads (`pageId`, `kind`, `updatedAt?`)
- Error Response Convention section with standard error codes and JSON examples (INVALID_INPUT, NOT_FOUND, CONFLICT, UNAUTHORIZED, INTERNAL)
- Phase 3 search contract extensions: `search.semantic`, `search.graphNeighbors`, `search.graphSuggestLinks`
- Phase 3 secondary analytics contracts:
  - `journal.query`, `journal.moodTrends`
  - `habits.getSummary`
  - `goals.getProgressSummary`
  - `finance.getSummary`
  - `travel.getItinerary`
  - `meals.getNutritionSummary`

### Changed
- `tasks.create` / `tasks.list` / `tasks.update` — `status` enum now explicitly includes `BLOCKED` (ADR-0008)
- `meals.create` — `type` enum documented: `BREAKFAST|LUNCH|DINNER|SNACK`
- Calendar/Schedule section reorganized: `calendar.getToday` added as the primary today-schedule command
- `travel.createTrip` request expanded to `title`, `destination`, `start_date`, `end_date`
- `travel.createCard` request now uses `trip_id`, `kind`, `title`, `props?`
- `habits.toggle` request key updated to `page_id`
- `finance.importCsv` now supports either `{ filename, content }` or `{ account_id, rows[] }` and returns created transactions

### Deprecated (bridge commands per ADR-0007)
- `schedule.getToday` — superseded by `calendar.getToday`; bridge mapping table updated with strikethrough
- `schedule.addTask` — superseded by `calendar.addEvent(type: "task")`; bridge mapping table updated with strikethrough
