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
> | `schedule.getToday` | `collection_query(collection_id: "col_calendar", filters: [start = today])` | Per ADR-0007, ScheduleItem is eliminated |
> | `schedule.addTask` | `vault_create_page(kind: "event", type: "task", ...)` | Per ADR-0007 |
>
> The same mapping applies to all other domain commands (projects, journal, habits, goals, meals, workouts, travel, finance). See `docs/CONVENTIONS.md` for the canonical naming convention.

---

## Command Matrix

### Tasks

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `tasks.create` | Create a new task | `title` (string), `description?`, `due_date?`, `project_id?`, `priority?` (enum), `type?`, `tags?` (string[]) | `Task` | CreateTaskModal, CommandPalette, AI agent `addTask` | `TaskService::create` |
| `tasks.list` | List tasks with filters | `status?` (enum), `project_id?`, `search?` | `Task[]` | TasksIndex, TodayDashboard, ProjectDetail | `TaskService::list` |
| `tasks.update` | Update a task | `id`, any updatable Task fields | `Task` | TaskDetailModal, TasksIndex (drag), TodayDashboard | `TaskService::update` |
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
| `travel.createTrip` | Create a trip folder | `destination` | `Trip` | Travel "New Trip" | `TravelService::create_trip` |
| `travel.createCard` | Add card to trip | `trip_path`, `title` | `Note` | Travel detail "Add Card" | `TravelService::create_card` |

### Finance

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `finance.getAccounts` | List manual accounts | — | `ManualAccount[]` | Finance Accounts tab | `FinanceService::get_accounts` |
| `finance.addAccount` | Add manual account | `name`, `type`, `balance` | `ManualAccount` | Finance "Add Account" | `FinanceService::add_account` |
| `finance.getBudget` | Get budget months | `month?` | `YNABBudgetMonth[]` | Finance Budget tab (Recharts) | `FinanceService::get_budget` |
| `finance.listTransactions` | List transactions | `month?` | `Transaction[]` | Finance Transactions tab | `FinanceService::list_transactions` |
| `finance.importCsv` | Import CSV file | `filename`, `content` | `void` | Finance Import tab | `FinanceService::import_csv` |

### Journal

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `journal.create` | Add journal entry | `date?`, `content`, `mood?`, `tags?` | `JournalEntry` | Journal view | `JournalService::create` |
| `journal.list` | List entries | `date_range?`, `mood?`, `tag?` | `JournalEntry[]` | Journal view | `JournalService::list` |

### Habits

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `habits.list` | List all habits | — | `Habit[]` | Habits view, TodayDashboard | `HabitService::list` |
| `habits.create` | Create a habit | `title`, `frequency?` | `Habit` | Habits "Add Habit" | `HabitService::create` |
| `habits.toggle` | Toggle completion | `id`, `date` | `Habit` | Habits daily check, TodayDashboard | `HabitService::toggle_completion` |

### Goals

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `goals.list` | List goals | `status?`, `project_id?` | `Goal[]` | Goals view | `GoalService::list` |
| `goals.create` | Create a goal | `title`, `description?`, `type`, `target_date`, `project_id?` | `Goal` | Goals "Add Goal", AI agent `addGoal` | `GoalService::create` |
| `goals.update` | Update a goal | `id`, updatable fields | `Goal` | Goals progress slider | `GoalService::update` |

### Meals / Recipes

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `meals.list` | List meals | `date_range?` | `Meal[]` | Meals view | `MealService::list` |
| `meals.create` | Log a meal | `date`, `type`, `description`, `recipe_id?`, `calories?` | `Meal` | Meals view | `MealService::create` |
| `recipes.list` | List recipes | `tags?` | `Recipe[]` | Meals recipe library | `RecipeService::list` |
| `recipes.create` | Create a recipe | `title`, `ingredients`, `instructions`, `calories?`, `tags?`, `image_url?` | `Recipe` | Meals recipe form | `RecipeService::create` |

### Workouts

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `workouts.list` | List workouts | — | `Workout[]` | Workouts view | `WorkoutService::list` |
| `workouts.create` | Log a workout | `name`, `date`, `exercises`, `duration` | `Workout` | Workouts view (planned) | `WorkoutService::create` |

### Calendar / Schedule

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `calendar.getWeek` | Get week events | `start_date?` | `CalendarEvent[]` | WeekDashboard | `CalendarService::get_week` |
| `calendar.addEvent` | Create event | `title`, `start`, `end`, `type`, `color?`, `description?`, `location?`, `linked_note_id?`, `task_id?` | `CalendarEvent` | WeekDashboard click-to-add | `CalendarService::add_event` |
| `calendar.updateEvent` | Update event | `id`, updatable fields | `CalendarEvent` | WeekDashboard drag | `CalendarService::update_event` |
| `calendar.deleteEvent` | Delete event | `id` | `void` | WeekDashboard right-click | `CalendarService::delete_event` |
| `schedule.getToday` | Get today's schedule | — | `ScheduleItem[]` | TodayDashboard timeline | `ScheduleService::get_today` |
| `schedule.addTask` | Add task to schedule | `task_id`, `start_time`, `duration_minutes` | `ScheduleItem` | TodayDashboard | `ScheduleService::add_task` |

### Search

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `search.global` | Search all content | `query`, `limit?` | `SearchResult[]` | CommandPalette, AI agent `searchBrain` | `SearchService::global` |

### Quick Capture

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `capture.save` | Save quick capture | `text` | `void` | TodayDashboard capture widget | `CaptureService::save` |

### AI (Phase 4 — currently frontend-direct)

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `ai.getSettings` | Get AI config | — | `AISettings` | Settings page | `AIService::get_settings` |
| `ai.setSettings` | Update AI config | Partial `AISettings` fields | `AISettings` | Settings page | `AIService::update_settings` |
| `ai.getModels` | List available models | — | `AIModel[]` | Settings, RightDrawer model picker | `AIService::list_models` |
| `ai.chat` | Send chat message | `model_id?`, `history`, `message`, `enable_agent?` | `ChatResponse { text, tool_results? }` | RightDrawer AI panel | `AIService::chat` |
| `ai.summarize` | Summarize a note | `note_id` | `string` | RightDrawer note summary | `AIService::summarize` |
| `ai.generateImage` | Generate image | `prompt` | `string` (base64 data URI) | ProjectDetail artifacts | `AIService::generate_image` |
| `ai.transcribe` | Transcribe audio | `audio_base64` | `string` (transcribed text) | RightDrawer voice input | `AIService::transcribe` |
| `ai.generateSpeech` | Text to speech | `text`, `voice?` | `string` (base64 audio) | RightDrawer auto-speak | `AIService::generate_speech` |
| `ai.validateKey` | Validate API key | `provider`, `key` | `boolean` | Settings key validation | `AIService::validate_key` |

## How to use this matrix

* **Frontend developers:** Import the generated IPC client methods (e.g. `tasks.create`) instead of hardcoding command names. Refer to this table to understand required fields and expected responses.
* **Backend developers:** Ensure that each command is implemented in the IPC layer (`ipc` crate) and forwards to the correct service. Match request/response types with the contracts.
* **Architects and reviewers:** Use this table during code reviews to verify that all commands are properly wired and documented.
