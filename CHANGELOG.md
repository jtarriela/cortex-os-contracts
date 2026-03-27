# Contracts Changelog

All notable changes to the IPC contract surface are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [Unreleased]

### Added
- Metadata-first list query contract surface:
  - `collection_query_summary`
  - `project_milestones_list`
  - `vault_list_summary`
- View-model transport tranche for the phase 0-3 performance refactor:
  - `tasks_list_view`
  - `projects_list_view`
  - `notes_tree_view`
  - `calendar_occurrences`
  - `search_query`
  - `page_detail`
  - `page_mutate`
  - `view_projection_rebuild`
- Combined page mutation command:
  - `vault_update_page`
- Cookbook contract surface for ADR-0039:
  - `recipes.get`
  - `recipes.importPreview`
  - `recipes.importCommit`
- Phase 6 checked-in binding source/generator for the hot-surface transport reset:
  - `contracts/specs/phase06-view-bindings.json`
  - `contracts/scripts/generate-phase06-bindings.mjs`
- Google explicit-convert contract surface:
  - `integrations.convertGoogleEventToTask` (`{ pageId } -> { taskId, calendarEventId, linkedNoteId?, alreadyConverted }`)
- Integration settings additive field:
  - `mirrorMigrationV2Done`

### Changed
- Phase 7 contracts docs now freeze the ADR-0043 Phase 6 view transports as the canonical hot-surface path:
  - `tasks.list` now documents `tasks_list_view` + `page_detail` for migrated hot-surface consumers
  - `projects.list` now documents `projects_list_view` + `page_detail`
  - `vault.getRoot` now documents `notes_tree_view`
  - `collection_query_summary` and `vault_list_summary` remain documented only as retained generic metadata helpers outside the migrated hot path
- `tasks.list` docs now point to summary-list reads plus `vault_read(page_id)` for full task detail hydration.
- `tasks.list` docs now also define `tasks_list_view` as the paginated performance-first task list transport, with `TaskListRow[]` semantics instead of generic collection summaries.
- `projects.list` docs now point to summary-list reads plus `vault_read(page_id)` for full project detail hydration.
- `projects.list` docs now also define `projects_list_view` as the paginated project-card transport.
- `vault.getRoot` docs now point to metadata-only vault tree reads instead of full page hydration.
- `vault.getRoot` docs now also define `notes_tree_view` as the paginated notes explorer transport.
- `vault.getFileContent` docs now point to `vault_read(page_id)` for explicit note body hydration.
- `vault.getFileContent` docs now point to `page_detail(page_id)` as the new canonical detail-open alias.
- `tasks.update` and `projects.update` docs now point to the combined `vault_update_page` mutation path.
- `page_created`, `page_updated`, and `page_deleted` payload docs now include additive `changeType` and `projectionKinds[]` metadata for narrower frontend invalidation.
- Phase 6 transport reset keeps the hot-surface command names stable but changes the generated response contracts:
  - `calendar_occurrences` now returns `ViewPage<CalendarOccurrenceRow>` instead of paginated `Page` rows
  - `search_query` now returns `ViewPage<SearchHitRow>` instead of paginated `Page` rows
  - `tasks_list_view`, `projects_list_view`, and `notes_tree_view` are documented against `ViewPage<SummaryViewRow>`
- Hot-surface paging docs now standardize on `ViewPage<T> { items, nextCursor, totalApprox?, snapshotToken }`, where:
  - `nextCursor` is opaque backend-issued paging state
  - `snapshotToken` changes when query/range membership or ordering changes
  - snapshot mismatch recovery is restart-from-first-page for the affected query/range, not suffix replacement against a stale cursor chain
  - `search_query` snapshot tokens also cover ranked membership/order changes for the active query
  - consumers must reset append state on snapshot mismatch instead of blindly appending
- `page_created`, `page_updated`, and `page_deleted` payload docs now define `effects[]` projection-impact metadata as the Phase 6 source of truth:
  - `effects[]` entries are `{ projection, impact }`
  - `projection ∈ { tasks, projects, notes, calendar, search }`
  - `impact ∈ { membership, order, summary, detail }`
  - `effects[]` are projection-membership based rather than kind-only, so a page can invalidate projections other than its own `kind`
  - `projection="calendar"` is narrowed to what `calendar_occurrence_projection` actually serves today
  - same-page writes inside the backend debounce window may be merged into one emitted event, so consumers must treat the delivered event as the latest merged invalidation unit
- `search_query` docs now state that pagination traverses the full ranked result set for the current snapshot instead of stopping at a fixed candidate window.
- Projection-backed view commands now document an explicit repair path via `view_projection_rebuild`, which rebuilds the Tasks/Projects/Notes/Calendar hot-surface read models from canonical `pages` rows.
- Contracts/codegen docs now describe the checked-in Phase 6 generated binding surface (`phase06-bindings.ts`) used by the frontend for the hot-surface transport family, replacing the superseded Phase 3 slice.
- Phase 7 cleanup docs now treat `tasks_list_view`, `projects_list_view`, and `notes_tree_view` as the canonical hot-surface list/tree transport, while `collection_query_summary` and `vault_list_summary` are retained only as generic metadata helpers outside the migrated hot path.
- `projects.milestones.list` docs now point to a backend-filtered summary query instead of collection-wide client filtering.
- Meals/Recipes contract docs now reflect the Cookbook split:
  - `recipes.list` now accepts backend-side filter/sort request fields and returns `RecipeCardSummary[]`
  - `recipes.create` / `recipes.update` now persist structured recipe sections and metadata instead of flat `ingredients[]` + `instructions`
  - `recipes.delete` now documents cookbook ownership instead of the legacy Meals recipe card flow
  - recipe import preview/commit semantics, deterministic markdown bodies, lazy legacy normalization, dedupe rules, and multimodal fallback are now documented
- Google editable-calendar policy now uses explicit conversion (no automatic inbound task mirroring):
  - `editableCalendars` now marks conversion/writeback eligibility only
  - `integrations.deleteMirroredEvent` now handles unconverted Google event pages and converted task bundles

## [0.10.13] — 2026-03-06

### Added
- RAG platform control-plane contract surface:
  - `rag_variant_list`
  - `rag_variant_create`
  - `rag_variant_clone`
  - `rag_variant_archive`
  - `rag_variant_set_active`
  - `rag_variant_set_shadow`
  - `rag_index_artifact_list`
  - `rag_index_artifact_build`
  - `rag_index_artifact_status`
  - `rag_index_artifact_promote`
  - `rag_index_artifact_rollback`
  - `rag_lab_run`

### Changed
- RAG trace/dashboard docs now define Phase 5/6 additive lineage and rollout metadata:
  - `RagTraceDetail.lineage` is now a structured lineage object with role/peer/comparison fields.
  - `RagFailureRow` now carries optional variant/artifact/lineage attribution.
  - `RagDashboardSummary` now documents an optional `live_experiment` aggregate block for active-vs-shadow deltas.
- RAG platform docs now define immutable variant/artifact snapshot semantics, build-job identity (`job_id == index_artifact_id` in v1), and shadow-sampling configuration defaults.

## [0.10.12] — 2026-03-06

### Added
- RAG eval / judge foundation contract surface:
  - `rag_eval_suite_list`
  - `rag_eval_run_start`
  - `rag_eval_run_status`
  - `rag_eval_run_results`
  - `rag_eval_baseline_set`
  - `rag_eval_case_export_from_trace`
  - `rag_judge_config_list`
  - `rag_judge_config_create`
  - `rag_judge_config_set_active`
  - `rag_judge_calibrate`

### Changed
- RAG docs now define Phase 4 deterministic eval payloads, baseline comparison rows, and redaction-by-default trace export semantics.
- Judge config docs now treat local calibration anchors as first-class request payloads for immutable config creation.

## [0.10.11] — 2026-03-06

### Added
- RAG dashboard contract surface:
  - `rag_dashboard_summary`
  - `rag_dashboard_failures`

### Changed
- `ai_chat` docs now state that `request_id` is the stable public trace identifier used by the RightDrawer RAG inspector and dashboard drilldowns.
- `rag_trace_get` / `rag_feedback_submit` docs now cover shared `ai_chat` + `ai_rag_query` trace usage.

## [0.10.10] — 2026-03-05

### Added
- RAG observability/refinement contract surface:
  - `rag_trace_get`
  - `rag_feedback_submit`

### Changed
- `ai_rag_query` now documents additive routed-retrieval metadata:
  - `retrieval_mode`
  - `query_type`
  - `citations[]`
- RAG/tooling docs now define trace-detail payload shape and hybrid chunk retriever source labels (`chunks_fts`, `vec_chunks`, `graph_edges`).

## [0.10.9] — 2026-02-25

### Added
- Vault markdown metadata parity maintenance contract surface:
  - `vault_markdown_metadata_audit`
  - `vault_markdown_metadata_repair`
- Vault workbench native open helper:
  - `vault_open_in_native_editor`

### Changed
- Vault docs now define Cortex-managed markdown frontmatter parity expectations (user-visible metadata), explicit audit/repair semantics, and linked Obsidian exclusion from canonical rewrite.

## [0.10.8] — 2026-02-24

### Added
- AI class planning + review apply contract surface:
  - `ai_class_project_plan_preview`
  - `review_apply`

### Changed
- `ai_chat` tool-result docs now include typed UI-action variants:
  - `request_scope_selection`
  - `open_review_item`
  - `context_used`
- `ReviewQueueItem` docs now include additive optional apply lifecycle metadata:
  - `applyStatus`
  - `appliedAt`
  - `applyError`
  - `resultJson`
- AI docs now define folder-scoped source selection semantics for class planning, including scoped linked Obsidian inclusion (`include_sources`, `obsidian_link_ids`) and compatibility-safe additive response fields.

## [0.10.7] — 2026-02-24

### Changed
- Travel Stage 1-2 docs now define stricter backend validation semantics for `travel.createItem` / `travel.updateItem` (date/time formats, `start_time < end_time`, positive `order_index`, non-empty `item_type`, and optional flight/lodging field format validation when present).
- Travel Stage 1-2 docs now define stricter backend validation semantics for `travel.createExpense` / `travel.updateExpense` (finite non-negative `amount`, optional `date`/`currency` format checks, and same-trip ownership validation for linked `location_id` / `item_id`).
- `travel.pushItemsToCalendar` frontend-usage notes now explicitly document selected-item push UX (existing `itemIds[]` contract support; no command-surface change).

## [0.10.6] — 2026-02-24

### Added
- Travel v2 Stage 6 calendar push contract surface:
  - `travel.pushItemsToCalendar`

### Changed
- Travel Stage 6 docs now define one-way Travel -> Calendar projection semantics (no reverse sync in v1), idempotent re-push behavior (`overwriteExisting`), and structured batch result reporting (`created`/`updated`/`skipped`/`errors` + row statuses).
- Travel docs now record additive linkage props for Stage 6 calendar projection (`trip_item.calendar_event_id`, `calendar_event.travel_*` metadata) and optional `syncExternal` outbound-mirror intent for Travel-generated local events.

## [0.10.5] — 2026-02-23

### Added
- Travel v2 Stage 5 planner AI contract surface:
  - `travel.aiSuggestIdeas`
  - `travel.optimizeDayPlanPreview`

### Changed
- Travel Stage 5 docs now define preview-only planner semantics with explicit frontend apply via standard Travel mutation commands (no `travel.aiApply*` command in v1).
- `travel.optimizeDayPlanPreview` docs now define `reorder_first_v1` response semantics, typed `changes[]` union (`reorder` + informational `note` rows), stale-preview `applyGuard`, and `degraded_deterministic` rationale-fallback behavior.

## [0.10.4] — 2026-02-23

### Added
- Travel v2 Stage 4 import contract surface:
  - `travel.importPreview`
  - `travel.importCommit`
  - `travel.gmailScanPreview`
  - `travel.gmailImportCommit`

### Changed
- `integrations.googleAuth` docs now include additive optional `scopeProfile` (`calendar` default, `calendar_gmail` for Travel Gmail scope upgrade) while preserving backward compatibility.
- `IntegrationSettings` docs now include `googleGmailConnected` to represent shared Google auth Gmail-scope availability for Travel Stage 4B.
- Travel import docs now define preview-only no-write semantics (`travel.importPreview`, `travel.gmailScanPreview`) and structured-data-first Gmail storage/indexing policy (raw message bodies/attachments not stored or indexed by default).

## [0.10.3] — 2026-02-23

### Added
- Travel v2 Stage 3 maps/routing/export contract surface:
  - `travel.routeComputeLeg`
  - `travel.routeComputeDay`
  - `travel.exportGoogleMaps`
  - `travel.resolveMapWaypoints`
  - `travel.getMapsProviderStatus`
  - `travel.getMapsJsConfig`

### Changed
- Travel routing/export docs now define explicit graceful fallback semantics for unsupported `travel.exportGoogleMaps.target = "saved_list_experimental"` responses (no silent failures).
- Travel Stage 3 docs now record transit day-route stitching behavior and route-cache observability metadata expectations (`cache` / `cacheStats` response fields).
- Travel v2 page-prop notes expanded for additive map/routing props (`map_lat`, `map_lng`, `map_query`, `map_formatted_address`, `google_place_id`, `map_resolved_at`, `map_resolution_source`, `route_exclude`).

## [0.10.2] — 2026-02-23

### Changed
- Travel v2 Stage 1-2 integrity patch updates:
  - `travel.moveItem` adds optional `clearLocation` to explicitly clear `location_id` without overloading omitted/null semantics
  - backend now validates target location kind/trip ownership for travel item create/move operations
  - legacy migration skips duplicates using `legacy_card_id`

### Notes
- `travel.moveItem` remains metadata-only in Stage 1-2 (filesystem relocation deferred)
- New location-scoped item creates use a stable per-location folder subdir (hybrid path strategy; no mass migration)

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
- ADR-0032 Habits Weekly Review calendar-sync contract:
  - `habits.syncWeekPlan` / `habits_sync_week_plan`
  - request shape: `{ reviewId }`
  - response: `HabitWeekSyncResult` (`created`, `updated`, `deleted`, `skipped`, `eventIds[]`)

### Changed
- `tasks.create` / `tasks.list` / `tasks.update` — `status` enum now explicitly includes `BLOCKED` (ADR-0008)
- `meals.create` — `type` enum documented: `BREAKFAST|LUNCH|DINNER|SNACK`
- Calendar/Schedule section reorganized: `calendar.getToday` added as the primary today-schedule command
- `travel.createTrip` request expanded to `title`, `destination`, `start_date`, `end_date`
- `travel.createCard` request now uses `trip_id`, `kind`, `title`, `props?`
- `habits.toggle` request key updated to `page_id`
- `finance.importCsv` now supports either `{ filename, content }` or `{ account_id, rows[] }` and returns created transactions
- `habits.toggle` now documents optional `completionType` (`standard|mvh`) with same-day mutual exclusivity semantics across `completed_dates` and `mvh_dates` (ADR-0032).
- `habits.getSummary` now documents split `STANDARD`/`MVH` metrics while preserving legacy combined totals (`completions`, `windowCompletions`) for compatibility.
- `calendar.rescheduleEvent` / `calendar.deleteEvent` now document linked habit-generated event write-back semantics for ADR-0032 Weekly Review plan sync.

### Deprecated (bridge commands per ADR-0007)
- `schedule.getToday` — superseded by `calendar.getToday`; bridge mapping table updated with strikethrough
- `schedule.addTask` — superseded by `calendar.addEvent(type: "task")`; bridge mapping table updated with strikethrough
