# IPC Wiring Matrix

This matrix documents the public **IPC contract** for Cortex OS. Each command listed here is defined in the contracts crate (Rust) and has a corresponding TypeScript client generated for the frontend. The purpose of this matrix is to ensure that **frontend**, **contracts** and **backend** stay in sync: when adding a command in one layer, update this document and open paired pull requests in `cortex-os-frontend` and `cortex-os-backend`.

**Source of truth:** Commands are derived from the frontend `dataService.ts` and `aiService.ts` API surface. Every async function the frontend calls will become an IPC command when the Tauri backend is wired.

## Naming Convention Reconciliation

> **Note (ADR-0006, ADR-0007):** The command names below use dot-notation with domain-specific namespaces (e.g., `tasks.create`, `journal.list`). Per ADR-0006, the production backend uses the EAV/Page model where all entities are pages. The target-state IPC surface uses **snake_case page-centric commands** as defined in `001_architecture.md` Section 6.2 (e.g., `vault_create_page`, `collection_query`, `page_update_props`).
>
> The domain-specific commands below are **Phase 0 bridge commands** — they document the frontend's current `dataService.ts` API surface and will be mapped to page-centric commands during Phase 1 IPC wiring:
>
> | Bridge Command | Target Command | Notes |
> |---|---|---|
> | `tasks.create` | `vault_create_page(kind: "task", ...)` | Properties normalized to EAV |
> | `tasks.list` | `collection_query(collection_id: "col_tasks", ...)` | Filters via EAV properties |
> | `tasks.update` | `page_update_props(page_id, props)` | Property updates via EAV |
> | `tasks.delete` | `vault_delete(page_id)` | Removes .md file + index |
> | ~~`schedule.getToday`~~ | `calendar.getToday` | **Removed** — ScheduleItem eliminated per ADR-0007; use `calendar.getToday` |
> | ~~`schedule.addTask`~~ | `calendar.addEvent(type: "task")` | **Removed** — replaced by `calendar.addEvent` per ADR-0007 |
>
> The same mapping applies to all other domain commands (projects, journal, habits, goals, meals, workouts, travel, finance). See `docs/CONVENTIONS.md` for the canonical naming convention.

---

## Command Matrix

### Tasks

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `tasks.create` | Create a new task | `title` (string), `description?`, `due_date?`, `project_id?`, `priority?` (enum: `HIGH\|MEDIUM\|LOW\|NONE`), `status?` (enum: `TODO\|DOING\|BLOCKED\|DONE\|ARCHIVED`), `type?`, `tags?` (string[]) | `Task` | CreateTaskModal, CommandPalette, AI agent `addTask` | `TaskService::create` |
| `tasks.list` | List tasks with filters | `status?` (enum: `TODO\|DOING\|BLOCKED\|DONE\|ARCHIVED`), `project_id?`, `search?` | `Task[]` | TasksIndex, TodayDashboard, ProjectDetail | `TaskService::list` |
| `tasks.update` | Update a task | `id`, any updatable Task fields incl. `status` (accepts `BLOCKED`), `sync_external?` (boolean) | `Task` | TaskDetailModal, TasksIndex (drag), TodayDashboard | `TaskService::update` |
| `tasks.delete` | Delete a task | `id` | `void` | TaskDetailModal | `TaskService::delete` |

### Projects

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `projects.create` | Create a project | `template_id?`, `title`, `description?`, `priority?` | `Project` | ProjectsIndex (new project) | `ProjectService::create` |
| `projects.list` | List projects | `status?`, `search?` | `Project[]` | ProjectsIndex | `ProjectService::list` |
| `projects.get` | Get project details | `id` | `ProjectDetail` | ProjectDetail view | `ProjectService::get_detail` |
| `projects.update` | Update project | `id`, updatable fields incl. `milestones`, `artifacts` | `Project` | ProjectDetail (milestones, artifacts) | `ProjectService::update` |

### Notes / Vault

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `vault.getRoot` | Get vault file tree | — | `FileNode[]` | NotesLibrary file tree | `VaultService::get_root` |
| `vault.getFileContent` | Get note content | `id` | `Note` | NotesLibrary note viewer, RightDrawer | `VaultService::get_file_content` |
| `notes.create` | Create a note | `title`, `content`, `path`, `tags?` | `Note` | NotesLibrary | `NoteService::create` |
| `notes.update` | Update a note | `id`, `title?`, `content?`, `tags?` | `Note` | NotesLibrary | `NoteService::update` |

### Travel

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `travel.listTrips` | List all trips | — | `Trip[]` | Travel gallery | `TravelService::list` |
| `travel.createTrip` | Create a trip folder | `title`, `destination`, `start_date`, `end_date` | `Trip` | Travel "New Trip" | `travel_create_trip` |
| `travel.createCard` | Add card to trip | `trip_id`, `kind`, `title`, `props?` | `Note` | Travel detail "Add Card" | `travel_create_card` |
| `travel.getItinerary` | Get trip + child cards | `trip_id` | `{ trip, cards[] }` | Travel itinerary detail | `travel_get_itinerary` |

### Finance

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `finance.getAccounts` | List manual accounts | — | `ManualAccount[]` | Finance Accounts tab | `FinanceService::get_accounts` |
| `finance.addAccount` | Add manual account | `name`, `type`, `balance` | `ManualAccount` | Finance "Add Account" | `FinanceService::add_account` |
| `finance.getBudget` | Get budget months | `month?` | `YNABBudgetMonth[]` | Finance Budget tab (Recharts) | `FinanceService::get_budget` |
| `finance.getSummary` | Get month rollup | `month?` | `FinanceSummary` | Finance drill-down metrics | `finance_get_summary` |
| `finance.listTransactions` | List transactions | `month?` | `Transaction[]` | Finance Transactions tab | `FinanceService::list_transactions` |
| `finance.importCsv` | Import CSV file | `filename`, `content` **or** `account_id`, `rows[]` | `Transaction[]` | Finance Import tab | `finance_import_csv` |

### Journal

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `journal.create` | Add journal entry | `date?`, `content`, `mood?`, `tags?` | `JournalEntry` | Journal view | `JournalService::create` |
| `journal.list` | List entries | `date_range?`, `mood?`, `tag?` | `JournalEntry[]` | Journal view | `JournalService::list` |
| `journal.query` | Filter entries by date/mood | `start_date?`, `end_date?`, `mood?` | `JournalEntry[]` | Journal timeline filtering | `journal_query` |
| `journal.moodTrends` | Aggregate mood counts | `start_date?`, `end_date?` | `{ mood, count }[]` | Journal mood chart | `journal_mood_trends` |

### Habits

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `habits.list` | List all habits | — | `Habit[]` | Habits view, TodayDashboard | `HabitService::list` |
| `habits.create` | Create a habit | `title`, `frequency?` | `Habit` | Habits "Add Habit" | `HabitService::create` |
| `habits.toggle` | Toggle completion | `page_id`, `date` | `Habit` | Habits daily check, TodayDashboard | `habits_toggle` |
| `habits.getSummary` | Habit analytics | `days?` | `HabitSummary[]` | Habits analytics panel | `habits_get_summary` |

### Goals

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `goals.list` | List goals | `status?`, `project_id?` | `Goal[]` | Goals view | `GoalService::list` |
| `goals.create` | Create a goal | `title`, `description?`, `type`, `target_date`, `project_id?` | `Goal` | Goals "Add Goal", AI agent `addGoal` | `GoalService::create` |
| `goals.update` | Update a goal | `id`, updatable fields | `Goal` | Goals progress slider | `GoalService::update` |
| `goals.getProgressSummary` | Goal rollup metrics | `project_id?` | `GoalProgressSummary` | Goals dashboard chart | `goals_get_progress_summary` |

### Meals / Recipes

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `meals.list` | List meals | `date_range?` | `Meal[]` | Meals view | `MealService::list` |
| `meals.create` | Log a meal | `date`, `type` (enum: `BREAKFAST\|LUNCH\|DINNER\|SNACK`), `description`, `recipe_id?`, `calories?` | `Meal` | Meals weekly planner slot | `MealService::create` |
| `meals.update` | Update a meal | `id`, updatable Meal fields | `Meal` | Meals weekly planner | `MealService::update` |
| `meals.delete` | Remove a meal | `id` | `void` | Meals planner slot replace | `MealService::delete` |
| `meals.getNutritionSummary` | Date-window nutrition rollup | `start_date?`, `end_date?` | `MealsNutritionSummary` | Meals analytics cards | `meals_get_nutrition_summary` |
| `recipes.list` | List recipes | `tags?` | `Recipe[]` | Meals recipe library | `RecipeService::list` |
| `recipes.create` | Create a recipe | `title`, `ingredients`, `instructions`, `calories?`, `tags?`, `image_url?` | `Recipe` | Meals recipe form | `RecipeService::create` |
| `recipes.update` | Update a recipe | `id`, updatable Recipe fields incl. `image_url` | `Recipe` | Meals recipe card (image upload) | `RecipeService::update` |
| `recipes.delete` | Delete a recipe | `id` | `void` | Meals recipe card (planned) | `RecipeService::delete` |

### Workouts

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `workouts.list` | List workouts | — | `Workout[]` | Workouts view | `WorkoutService::list` |
| `workouts.create` | Log a workout | `name`, `date`, `exercises`, `duration` | `Workout` | Workouts view (planned) | `WorkoutService::create` |

### Calendar / Schedule

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `calendar.getToday` | Get today's schedule events | — | `CalendarEvent[]` | TodayDashboard timeline | `CalendarService::get_today` |
| `calendar.getWeek` | Get week events | `start_date?` | `CalendarEvent[]` | WeekDashboard | `CalendarService::get_week` |
| `calendar.addEvent` | Create event | `title`, `start` (ISO datetime), `end` (ISO datetime), `type` (enum: `event\|task\|reminder\|deep-work`), `color?`, `description?`, `location?`, `linked_note_id?`, `task_id?` | `CalendarEvent` | TodayDashboard schedule, WeekDashboard click-to-add | `CalendarService::add_event` |
| `calendar.updateEvent` | Update event | `id`, updatable fields incl. `sync_external?` (boolean) | `CalendarEvent` | WeekDashboard drag | `CalendarService::update_event` |
| `calendar.deleteEvent` | Delete event | `id` | `void` | WeekDashboard right-click | `CalendarService::delete_event` |

### Integrations (Google Calendar)

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `integrations.googleAuth` | Authenticate with Google | — | `string` (auth URL) | Settings Integrations | `GoogleService::authenticate` |
| `integrations.googleCalendars` | List available calendars | — | `string[]` | Settings Integrations | `GoogleService::get_calendars` |
| `integrations.triggerSync` | Trigger two-way sync manually | — | `boolean` | Settings Integrations | `GoogleService::trigger_sync` |

### Search

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `search.global` | Search all content | `query`, `limit?` | `SearchResult[]` | CommandPalette, AI agent `searchBrain` | `SearchService::global` |
| `search.semantic` | Vector-only semantic search | `query`, `limit?` | `SearchResult[]` | Future semantic tab / agent tools | `search_semantic` |
| `search.graphNeighbors` | Traverse related pages by link distance | `page_id`, `depth?`, `limit?` | `SearchResult[]` | CommandPalette related graph badges | `search_graph_neighbors` |
| `search.graphSuggestLinks` | Suggest unlinked related pages | `page_id`, `limit?` | `SearchResult[]` | Link suggestion workflows (Phase 4 prep) | `search_graph_suggest_links` |

## Phase 3 Page-Centric Notes

Search commands are powered by:

- FTS5 lexical ranking (`pages_fts`)
- sqlite-vec chunk vectors (`search_chunk_vec`)
- bidirectional wikilink graph edges (`graph_edges`)

Embedding provider options:

- hash embeddings (default deterministic fallback)
- Ollama embeddings (`CORTEX_SEARCH_EMBED_PROVIDER=ollama`, optional local runtime)

### Quick Capture

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `capture.save` | Save quick capture | `text` | `void` | TodayDashboard capture widget | `CaptureService::save` |

### AI (Phase 4 — backend IPC foundation)

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `ai_get_models` | List available models | — | `AIModel[]` | Settings model picker, RightDrawer model selector | `ai_get_models` |
| `ai_chat` | Send chat message and emit stream events | `request: { history, message, model_id?, enable_agent? }` | `ChatResponse { text, tool_results[], request_id }` | RightDrawer AI panel | `ai_chat` |
| `ai_summarize` | Summarize note content | `request: { title, body }` | `string` | Note summary panel | `ai_summarize` |
| `ai_generate_image` | Generate image artifact | `request: { prompt }` | `string` (data URI) | Project artifact generation | `ai_generate_image` |
| `ai_transcribe` | Transcribe audio | `request: { audio_base64 }` | `string` | Voice input transcription | `ai_transcribe` |
| `ai_synthesize` | Text to speech | `request: { text, voice? }` | `string` (base64 WAV) | Chat auto-speak | `ai_synthesize` |
| `ai_validate_key` | Validate provider credential/endpoint format | `request: { provider, key }` | `boolean` | Settings provider verification | `ai_validate_key` |
| `review_list` | List Morning Review items | `status?` (`pending|approved|rejected`) | `ReviewQueueItem[]` | Morning Review UI | `review_list` |
| `review_approve` | Approve review item | `item_id`, `edited_json?` | `boolean` | Morning Review UI | `review_approve` |
| `review_reject` | Reject review item | `item_id` | `boolean` | Morning Review UI | `review_reject` |
| `token_usage` | Query token/cost usage rollups | `days?` | `TokenUsageEntry[]` | Settings usage dashboard | `token_usage` |

## Event Streams

Realtime frontend invalidation for Phase 3 uses Tauri event channels in addition to request/response commands.

| Event | Description | Payload fields | Frontend usage | Backend emitter |
|-------|-------------|----------------|----------------|-----------------|
| `page_created` | A page/entity was created | `pageId`, `kind`, `updatedAt?` | `stores/realtimeStore.ts` subscription; triggers view invalidation/remount in `App.tsx` | Page-creation handlers (`vault_create_page`, `capture_save`, etc.) |
| `page_updated` | A page/entity was updated | `pageId`, `kind`, `updatedAt?` | `stores/realtimeStore.ts` subscription; triggers view invalidation/remount in `App.tsx` | Page-update handlers (`page_update_props`, `page_update_body`, `habits_toggle`) |
| `page_deleted` | A page/entity was deleted | `pageId`, `kind`, `updatedAt?` | `stores/realtimeStore.ts` subscription; triggers view invalidation/remount in `App.tsx` | `vault_delete` |
| `ai_stream_chunk` | Streamed AI text chunk | `requestId`, `text` | `services/aiService.ts` streaming updates in RightDrawer | `ai_chat` |
| `ai_stream_done` | Stream completion metadata | `requestId`, `inputTokens`, `outputTokens`, `estimatedCostUsd` | `services/aiService.ts` completion handling | `ai_chat` |
| `ai_stream_error` | Stream failure notification | `requestId`, `error` | `services/aiService.ts` error fallback | Future provider stream handlers |

## Error Response Convention

All IPC commands return a tagged Result type. On error the response is a JSON object with `error` key:

```json
{ "error": { "code": "NOT_FOUND", "message": "Task id=abc not found" } }
```

Standard error codes:

| Code | HTTP analogue | When to use |
|------|--------------|-------------|
| `NOT_FOUND` | 404 | Requested resource does not exist |
| `INVALID_INPUT` | 400 | Required field missing or value out of range |
| `CONFLICT` | 409 | Duplicate ID or concurrent write conflict |
| `UNAUTHORIZED` | 401 | DB key rejected (wrong passphrase) |
| `INTERNAL` | 500 | Unexpected Rust panic / SQLite error |

**Example — `tasks.create` with missing required field:**

```json
// Request
{ "title": "" }

// Response (error)
{ "error": { "code": "INVALID_INPUT", "message": "title must not be empty" } }
```

**Example — `tasks.update` with unknown id:**

```json
// Request
{ "id": "task-does-not-exist", "status": "DONE" }

// Response (error)
{ "error": { "code": "NOT_FOUND", "message": "Task id=task-does-not-exist not found" } }
```

---

## How to use this matrix

* **Frontend developers:** Import the generated IPC client methods (e.g. `tasks.create`) instead of hardcoding command names. Refer to this table to understand required fields and expected responses.
* **Backend developers:** Ensure that each command is implemented in the IPC layer (`ipc` crate) and forwards to the correct service. Match request/response types with the contracts.
* **Architects and reviewers:** Use this table during code reviews to verify that all commands are properly wired and documented.
