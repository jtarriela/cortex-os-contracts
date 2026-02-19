# Contracts Changelog

All notable changes to the IPC contract surface are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

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
