# IPC Wiring Matrix

This matrix documents the public **IPC contract** for Cortex OS.  Each command listed here is defined in the contracts crate (Rust) and has a corresponding TypeScript client generated for the frontend.  The purpose of this matrix is to ensure that **frontend**, **contracts** and **backend** stay in sync: when adding a command in one layer, update this document and open paired pull requests in `cortex-os-frontend` and `cortex-os-backend`.

## Table structure

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `tasks.create` | Create a new task | `title` (string), `description?` (string), `due_date?` (ISO date), `project_id?` (string), `priority?` (enum), `tags?` (string[]) | `Task` object | Called from CreateTaskModal and command palette | `TaskService::create` via `#[tauri::command]` |
| `tasks.list` | List tasks with optional filters | `status?` (enum), `project_id?` (string), `search?` (string) | `Task[]` | Used by TasksIndex and Today/Week dashboards | `TaskService::list` |
| `tasks.update` | Update an existing task | `id` (string), any updatable fields | `Task` | Called when editing a task in detail views | `TaskService::update` |
| `tasks.delete` | Delete a task | `id` (string) | `void` | Called from task detail or index | `TaskService::delete` |
| `projects.create` | Create a project with optional milestones | `title` (string), `description?`, `priority?`, `date_range?`, `milestones?` | `Project` | Used by ProjectsIndex to add new projects | `ProjectService::create` |
| `projects.list` | List projects | `status?`, `search?` | `Project[]` | Used by ProjectsIndex | `ProjectService::list` |
| `projects.get` | Get project details | `id` (string) | `ProjectDetail` | Used by ProjectDetail | `ProjectService::get_detail` |
| `projects.update` | Update project | `id` (string), updatable fields | `Project` | Editing a project | `ProjectService::update` |
| `notes.create` | Create a note in the vault | `title`, `content`, `path`, `tags?` | `Note` | Used by NotesLibrary | `NoteService::create` |
| `notes.list` | List notes | `path?`, `search?` | `Note[]` | NotesLibrary and command palette search | `NoteService::list` |
| `notes.get` | Get note content | `id` | `Note` with `content` | Note detail drawer | `NoteService::get` |
| `notes.update` | Update a note | `id`, updatable fields | `Note` | Editing a note | `NoteService::update` |
| `travel.listTrips` | List trips | — | `Trip[]` | Travel view | `TravelService::list` |
| `finance.getBudget` | Get YNAB budget for a month | `month` (YYYY-MM) | `YNABBudgetMonth` | Finance view monthly summary | `FinanceService::get_budget` |
| `finance.listTransactions` | List transactions | `month` (YYYY-MM) | `Transaction[]` | Finance view transaction list | `FinanceService::list_transactions` |
| `journal.create` | Add a journal entry | `date` (ISO date), `content`, `mood?`, `tags?` | `JournalEntry` | Journal view | `JournalService::create` |
| `journal.list` | List journal entries | `date_range?`, `mood?`, `tag?` | `JournalEntry[]` | Journal view filters | `JournalService::list` |
| `habits.create` | Create a habit | `title`, `frequency` | `Habit` | Habits view | `HabitService::create` |
| `habits.toggle` | Toggle habit completion | `id`, `date` | `Habit` | Mark habit complete/incomplete | `HabitService::toggle_completion` |
| `goals.create` | Create a goal | `title`, `description?`, `type`, `target_date`, `project_id?` | `Goal` | Goals view | `GoalService::create` |
| `goals.list` | List goals | `status?`, `project_id?` | `Goal[]` | Goals view filters | `GoalService::list` |
| `meals.create` | Create a meal record | `date`, `type`, `description`, `recipe_id?`, `calories?` | `Meal` | Meals view | `MealService::create` |
| `meals.list` | List meals | `date_range?` | `Meal[]` | Meals view | `MealService::list` |
| `ai.getModels` | List available AI models | — | `AIModel[]` | Settings page to choose model【267591510078076†L218-L239】 | `AIService::list_models` |
| `ai.setSettings` | Update AI settings | Partial `AISettings` | `AISettings` | Settings page toggles (voice, auto speak, preferredVoice, etc.) | `AIService::update_settings` |
| `calendar.getDay` | Get a day view of events & tasks | `date` | `DayView` | TodayDashboard | `CalendarService::get_day` |
| `calendar.getWeek` | Get a week view | `start_date` | `WeekView` | WeekDashboard | `CalendarService::get_week` |

## How to use this matrix

* **Frontend developers:** Import the generated IPC client methods (e.g. `tasks.create`) instead of hardcoding command names.  Refer to this table to understand required fields and expected responses.
* **Backend developers:** Ensure that each command is implemented in the IPC layer (`ipc` crate) and forwards to the correct service.  Match request/response types with the contracts.
* **Architects and reviewers:** Use this table during code reviews to verify that all commands are properly wired and documented.
