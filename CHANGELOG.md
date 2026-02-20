# Contracts Changelog

All notable changes to the IPC contract surface are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

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
