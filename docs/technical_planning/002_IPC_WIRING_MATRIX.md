# IPC Wiring Matrix

This matrix documents the public **IPC contract** for Cortex OS. Each command listed here is defined in the contracts crate (Rust) and has a corresponding TypeScript client generated for the frontend. The purpose of this matrix is to ensure that **frontend**, **contracts** and **backend** stay in sync: when adding a command in one layer, update this document and open paired pull requests in `cortex-os-frontend` and `cortex-os-backend`.

**Source of truth:** Commands are derived from the frontend `dataService.ts` and `aiService.ts` API surface. Every async function the frontend calls will become an IPC command when the Tauri backend is wired.

## Naming Convention Reconciliation

> **Note (ADR-0006, ADR-0007):** The command names below use dot-notation with domain-specific namespaces (e.g., `tasks.create`, `journal.list`). Per ADR-0006, the production backend uses the EAV/Page model where all entities are pages. The target-state IPC surface uses **snake_case page-centric commands** as defined in `001_architecture.md` Section 6.2 (e.g., `vault_create_page`, `collection_query`, `page_update_props`).
>
> The domain-specific commands below are **Phase 0 bridge commands** — they document the frontend's `services/backend.ts` API surface mapped to EAV page-centric commands. **Phase 1 IPC wiring is complete** — all domain operations go through the generic EAV command surface:
>
> | Bridge Command | Target Command | Notes |
> |---|---|---|
> | `tasks.create` | `vault_create_page(kind: "task", ...)` | Properties normalized to EAV |
> | `tasks.list` | `collection_query(collectionId: "col_tasks")` | Filters via EAV properties |
> | `tasks.update` | `page_update_props(pageId, props)` | Property updates via EAV |
> | `tasks.delete` | `vault_delete(pageId)` | Removes .md file + index |
> | ~~`schedule.getToday`~~ | `calendar.getToday` | **Removed** — ScheduleItem eliminated per ADR-0007; use `calendar.getToday` |
> | ~~`schedule.addTask`~~ | `calendar.addEvent(type: "task")` | **Removed** — replaced by `calendar.addEvent` per ADR-0007 |
>
> The same mapping applies to all other domain commands (projects, journal, habits, goals, meals, workouts, travel, finance). See `docs/CONVENTIONS.md` for the canonical naming convention.

### Tauri Argument Casing (Critical)

Top-level command arguments passed from frontend `invoke()` use **camelCase** keys derived from Rust parameter names.

- `collection_query(collection_id: String)` → `invoke('collection_query', { collectionId })`
- `calendar_get_week(start_date: Option<String>)` → `invoke('calendar_get_week', { startDate })`
- `calendar_get_range(start_date: String, end_date: String)` → `invoke('calendar_get_range', { startDate, endDate })`
- `travel_create_trip(start_date, end_date, budget?)` → `invoke('travel_create_trip', { startDate, endDate, budget })`
- `habits_toggle(page_id)` → `invoke('habits_toggle', { pageId })`

Nested `request` payloads keep their documented serde field names (snake_case).

---

## Command Matrix

### Tasks

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `tasks.create` | Create a new task | `title` (string), `description?`, `due_date?`, `project_id?`, `priority?` (enum: `HIGH\|MEDIUM\|LOW\|NONE`), `status?` (enum: `TODO\|DOING\|BLOCKED\|DONE\|ARCHIVED`), `type?`, `tags?` (string[]) | `Task` | CreateTaskModal, CommandPalette, AI agent `addTask` | `vault_create_page(kind:"task", props, body)` |
| `tasks.list` | List tasks with filters | `status?` (enum: `TODO\|DOING\|BLOCKED\|DONE\|ARCHIVED`), `project_id?`, `search?` | `Task[]` | TasksIndex, TodayDashboard, ProjectDetail | `collection_query("tasks")` |
| `tasks.update` | Update a task | `id`, any updatable Task fields incl. `status` (accepts `BLOCKED`), `sync_external?` (boolean), planning fields (`planned_start_date`, `planned_end_date`, `baseline_start_date`, `baseline_end_date`, `actual_start_date`, `actual_end_date`), dependencies (`dependencies[]`: `{ predecessor_id, type:\"FS\", lag_days? }`) | `Task` | TaskDetailModal, TasksIndex (drag), TodayDashboard, Project Timeline | `page_update_props(page_id, props)` |
| `tasks.delete` | Delete a task | `id` | `void` | TaskDetailModal | `vault_delete(page_id)` |

### Projects

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `projects.create` | Create a project | `template_id?`, `title`, `description?`, `priority?` | `Project` | ProjectsIndex (new project) | `vault_create_page(kind:"project", props)` |
| `projects.list` | List projects | `status?`, `search?` | `Project[]` | ProjectsIndex | `collection_query("projects")` |
| `projects.get` | Get project details | `id` | `ProjectDetail` | ProjectDetail view | `vault_read(page_id)` |
| `projects.update` | Update project | `id`, updatable fields incl. status/priority/date-range, `artifacts`, `columns` (milestones are dual-write body + milestone pages) | `Project` | ProjectDetail (overview + timeline), ProjectsIndex cards | `page_update_props(page_id, props)` |

### Project Milestones (Dual-Write)

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `projects.milestones.list` | List milestone pages for project | `project_id` | `ProjectMilestone[]` | ProjectDetail timeline + body sync | `collection_query(\"col_project_milestones\")` + frontend filter by `project_ref` |
| `projects.milestones.create` | Create milestone page | `project_ref`, `title`, `target_date`, `status`, `dependencies[]`, `baseline_date?`, `completed_date?`, `checklist_anchor_id` | `ProjectMilestone` | Timeline add milestone, body→page sync | `vault_create_page(kind:\"project_milestone\", props)` |
| `projects.milestones.update` | Update milestone page | `id` + updatable milestone fields | `ProjectMilestone` | Timeline edits, body checkbox/status sync | `page_update_props(page_id, props)` |
| `projects.milestones.delete` | Delete milestone page | `id` | `void` | Timeline delete, body line delete sync | `vault_delete(page_id)` |

> Dual-write sync policy (ADR-0021):
> - Body checklist lines use stable anchors: `<!-- milestone:{checklist_anchor_id} -->`.
> - Sync executes on each project save and milestone mutation.
> - Conflict resolution: milestone page wins.
> - Removing a markerized checklist line deletes the linked milestone page.
> - Checkbox mapping: checked => `COMPLETED`, unchecked => `NOT_STARTED`.

### Notes / Vault

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `vault.getRoot` | Get vault file tree | — | `FileNode[]` | NotesLibrary file tree | `VaultService::get_root` |
| `vault.getFileContent` | Get note content | `id` | `Note` | NotesLibrary note viewer, RightDrawer | `VaultService::get_file_content` |
| `notes.create` | Create a note | `title`, `content`, `path`, `tags?` | `Note` | NotesLibrary | `NoteService::create` |
| `notes.update` | Update a note | `id`, `title?`, `content?`, `tags?` | `Note` | NotesLibrary | `NoteService::update` |
| `vault_get_profile` | Read active vault onboarding profile | — | `VaultProfile \| null` | App bootstrap gate | `vault_get_profile` |
| `vault_create` | Create/activate vault profile + starter structure | `request: { root_path, name? }` | `VaultProfile` | Vault Setup splash | `vault_create` |
| `vault_select` | Validate/select an existing vault profile | `request: { root_path }` | `VaultProfile` | Vault Setup splash | `vault_select` |
| `save_commit` | Persist note body and enqueue post-commit indexing | `request: { page_id, body }` | `Page` | Notes editor autosave/flush | `save_commit` |
| `index_queue_status` | Read indexing queue state for diagnostics/progress | `limit?` | `IndexQueueJob[]` | Debug/progress UI | `index_queue_status` |

### Canonical Page Mutations (Phase 5 alignment)

| Command | Description | Request fields | Response fields | Notes |
|---------|-------------|----------------|----------------|-------|
| `vault_create_page` | Create a page in EAV + vault markdown | `kind`, `props`, `body?` | `Page` | `props.title` is canonical title input. Optional top-level `title` is compatibility-only and not required by FE/contracts payloads. |
| `page_update_body` | Update page markdown body | `page_id`, `body` | `Page` | Persists DB body and rewrites markdown file under active vault root. |
| `save_commit` | Durable save+index operation | `request: { page_id, body }` | `Page` | Persists markdown first, then enqueues/coalesces indexing jobs. |
| `vault_delete` | Delete page and markdown | `page_id` | `void` | Deletes the corresponding markdown file path before DB removal. |

### Travel

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `travel.listTrips` | List all trips | — | `Trip[]` | Travel gallery | `TravelService::list` |
| `travel.createTrip` | Create a trip folder and overview note | `title`, `destination`, `startDate`, `endDate`, `budget?` | `Trip` | Travel "New Trip" splash modal (destination + dates + duration + optional budget) | `travel_create_trip` (`Travel/Trips/<slug>/Overview.md`, status normalized to `Planning`) |
| `travel.createCard` | Add card markdown note to trip | `tripId`, `kind`, `title`, `props?` | `Note` | Travel detail "Add Card" | `travel_create_card` (`Travel/Trips/<slug>/<card-title-slug>.md` with collision suffixing) |
| `travel.getItinerary` | Get trip + child cards | `tripId` | `{ trip, cards[] }` | Travel itinerary detail | `travel_get_itinerary` |

### Finance

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `finance.getAccounts` | List manual accounts | — | `ManualAccount[]` | Finance Accounts tab | `FinanceService::get_accounts` |
| `finance.addAccount` | Add manual account | `name`, `type`, `balance` | `ManualAccount` | Finance "Add Account" | `FinanceService::add_account` |
| `finance.getBudget` | Get budget months | `month?` | `YNABBudgetMonth[]` | Finance Budget tab (Recharts) | `FinanceService::get_budget` |
| `finance.getSummary` | Get month rollup | `month?` | `FinanceSummary` | Finance drill-down metrics | `finance_get_summary` |
| `finance.listTransactions` | List transactions | `month?` | `Transaction[]` | Finance Transactions tab | `FinanceService::list_transactions` |
| `finance.importCsv` | Import CSV file | `filename`, `content` **or** `account_id`, `rows[]` | `Transaction[]` | Finance Import tab | `finance_import_csv` |
| `finance.ynabStatus` | Read local YNAB connection/sync status | — | `FinanceYnabStatus` | Finance YNAB onboarding/status card | `finance_ynab_status` |
| `finance.ynabConnectPat` | Save + validate YNAB Personal Access Token and cache budgets | `token`, `selectedBudgetId?` | `{ status: FinanceYnabStatus }` | Finance YNAB connect flow | `finance_ynab_connect_pat` |
| `finance.ynabDisconnect` | Remove YNAB token and reset selected budget state (local synced data preserved) | — | `boolean` | Finance YNAB disconnect action | `finance_ynab_disconnect` |
| `finance.ynabSync` | Full/delta YNAB view-only sync to local `ynab_*` pages + CSV mirror + analytics refresh | `budgetId?`, `mode?` (`auto\\|full\\|delta`), `writeCsv?`, `reanalyze?` | `FinanceYnabSyncResult` | Finance YNAB sync controls | `finance_ynab_sync` |
| `finance.ynabGetTrackerConfig` | Read tracked-category analyzer config for selected or specified budget | `budgetId?` | `FinanceTrackedCategoryConfig` | Finance tracked categories manager | `finance_ynab_get_tracker_config` |
| `finance.ynabSaveTrackerConfig` | Persist tracked-category analyzer config | `config { budgetId, categories[] }` | `FinanceTrackedCategoryConfig` | Finance tracked categories manager | `finance_ynab_save_tracker_config` |
| `finance.ynabGetAnalytics` | Compute/read local YNAB whole-budget + tracked-category metrics | `monthKey?`, `budgetId?` | `FinanceYnabAnalytics` | Finance dashboard charts + category metrics | `finance_ynab_get_analytics` |

### Journal

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `journal.create` | Add journal entry | `date?`, `content`, `mood?`, `tags?` | `JournalEntry` | Journal view | `JournalService::create` |
| `journal.list` | List entries | `date_range?`, `mood?`, `tag?` | `JournalEntry[]` | Journal view | `JournalService::list` |
| `journal.query` | Filter entries by date/mood | `startDate?`, `endDate?`, `mood?` | `JournalEntry[]` | Journal timeline filtering | `journal_query` |
| `journal.moodTrends` | Aggregate mood counts | `startDate?`, `endDate?` | `{ mood, count }[]` | Journal mood chart | `journal_mood_trends` |

### Habits

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `habits.list` | List all habits | — | `Habit[]` | Habits view, TodayDashboard | `HabitService::list` |
| `habits.create` | Create a habit | `title`, `frequency?` | `Habit` | Habits "Add Habit" | `HabitService::create` |
| `habits.toggle` | Toggle completion | `pageId`, `date` | `Habit` | Habits daily check, TodayDashboard | `habits_toggle` |
| `habits.getSummary` | Habit analytics | `days?` | `HabitSummary[]` | Habits analytics panel | `habits_get_summary` |

### Goals

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `goals.list` | List goals | `status?`, `project_id?` | `Goal[]` | Goals view | `GoalService::list` |
| `goals.create` | Create a goal | `title`, `description?`, `type`, `target_date`, `project_id?` | `Goal` | Goals "Add Goal", AI agent `addGoal` | `GoalService::create` |
| `goals.update` | Update a goal | `id`, updatable fields | `Goal` | Goals progress slider | `GoalService::update` |
| `goals.getProgressSummary` | Goal rollup metrics | `projectId?` | `GoalProgressSummary` | Goals dashboard chart | `goals_get_progress_summary` |

### Meals / Recipes

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `meals.list` | List meals | `date_range?` | `Meal[]` | Meals view | `MealService::list` |
| `meals.create` | Log a meal | `date`, `type` (enum: `BREAKFAST\|LUNCH\|DINNER\|SNACK`), `description`, `recipe_id?`, `calories?` | `Meal` | Meals weekly planner slot | `MealService::create` |
| `meals.update` | Update a meal | `id`, updatable Meal fields | `Meal` | Meals weekly planner | `MealService::update` |
| `meals.delete` | Remove a meal | `id` | `void` | Meals planner slot replace | `MealService::delete` |
| `meals.getNutritionSummary` | Date-window nutrition rollup | `startDate?`, `endDate?` | `MealsNutritionSummary` | Meals analytics cards | `meals_get_nutrition_summary` |
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
| `calendar.getToday` | Get today's schedule events | — | `CalendarEvent[]` | TodayDashboard timeline | `calendar_get_today()` |
| `calendar.getWeek` | Get week events | `startDate?` | `CalendarEvent[]` | WeekDashboard | `calendar_get_week(start_date)` |
| `calendar.getRange` | Get calendar events in explicit date window | `startDate` (ISO datetime/date), `endDate` (ISO datetime/date) | `CalendarEvent[]` | `useCalendarWorkspace` range loading for DayFlow week/day/month views | `calendar_get_range(start_date, end_date)` (inclusive `startDate`, exclusive `endDate`, includes `calendar_event` + scheduled `task` pages, sorted by `COALESCE(props.start, props.start_time)`) |
| `calendar.addEvent` | Create event | `title`, `start` (ISO datetime), `end` (ISO datetime), `type` (enum: `event\|task\|reminder\|deep-work`), `color?`, `description?`, `location?`, `linked_note_id?`, `task_id?` | `CalendarEvent` | TodayDashboard schedule, WeekDashboard click-to-add | `vault_create_page(kind:"calendar_event", props)` |
| `calendar.updateEvent` | Update event | `id`, updatable fields incl. `sync_external?` (boolean) | `CalendarEvent` | WeekDashboard drag | `page_update_props(page_id, props)` |
| `calendar.deleteEvent` | Delete/unschedule event | `id` | `void` | WeekDashboard right-click/delete action | `calendar_delete_event(event_id)` (`calendar_event` delete, `task` unschedule path) |
| `calendar.scheduleTask` | Schedule an existing task by writing task time props | `taskId`, `start` (ISO datetime for timed; `"YYYY-MM-DD"` for all-day), `end` (ISO datetime or next-day date), `allDay` (bool) | `CalendarEvent` | WeekDashboard schedule modal (existing task path) | `calendar_schedule_task(task_id, start, end, all_day)` (updates `task.start_time/end_time`) |
| `calendar.rescheduleEvent` | Move a scheduled item to a new slot. Supports `calendar_event` and `task` IDs. Returns `INVALID_INPUT` if event is read-only (Google-sourced — FR-027) | `eventId`, `start`, `end`, `allDay` (bool) | `CalendarEvent` | DayFlow drag/resize mutation path | `calendar_reschedule_event(event_id, start, end, all_day)` |
| `calendar.eventIsEditable` | Query whether a calendar event may be mutated. Returns `false` for Google-sourced read-only events (FR-027). Use on page load to pre-populate editability flags without optimistic guessing. | `eventId` (string) | `boolean` | DayFlow adapter; `useCalendarWorkspace` on event load (E25) | `calendar_event_is_editable(event_id)` |

> **[E24] Drag/Drop Intent Mapping** (ADR-0018 External Drop Gate):
>
> | Slot type | `allDay` | `start` format | `end` format |
> |-|-|-|-|
> | Timed (week/day view) | `false` | full ISO datetime `"2026-02-20T14:00:00Z"` | full ISO datetime |
> | All-day (month view / all-day row) | `true` | date-only `"2026-02-20"` | next calendar date `"2026-02-21"` |
>
> `CalendarEvent` response shape includes `readOnly: boolean` (sourced from `props.read_only`). Frontend must use this field to gate drag/resize/edit interactions before initiating — snap-back is not the primary permission model (E25, FR-027). The backend enforces all mutation guards server-side independently.

> **[E25] Permission Policy — Mixed Editability** (ADR-0018 Mixed Editability Gate):
>
> | Event source | `readOnly` | Drag/resize | Edit props | Delete |
> |-|-|-|-|-|
> | Cortex-managed | `false` | allowed | allowed | allowed |
> | Google-sourced (inbound sync) | `true` | blocked at UI + backend | blocked at backend | blocked at backend |
>
> **`readOnly` field on `CalendarEvent`:**
> - Populated from `props.read_only: bool` in the backend EAV store.
> - Google-inbound sync sets `read_only = true`; Cortex-created events omit the field (defaults `false`).
> - Frontend normalises absent field to `false` in the adapter layer.
>
> **Error semantics for read-only mutation attempts:**
>
> Any IPC command that mutates a read-only calendar event (`calendar.rescheduleEvent`, `calendar.updateEvent`, `calendar.deleteEvent`) returns:
>
> ```json
> { "error": { "code": "INVALID_INPUT", "message": "event is read-only: cannot mutate Google-sourced events" } }
> ```
>
> Frontend must **not** reach this path in normal operation — the UI pre-flight guard (`canEditEvent(event)`) prevents the invoke. The backend guard is a defence-in-depth layer (FR-027).

> **[E26] Keyboard Interaction → IPC Command Mapping** (ADR-0018 A11y/Keyboard Gate, FR-018):
>
> Keyboard navigation routes through the same DayFlow plugin callbacks as pointer interactions. No new IPC commands are required. The `@dayflow/plugin-keyboard-shortcuts` plugin (already bootstrapped in `dayflowPlugins.ts`) fires the same `onEventUpdate`, `onEventCreate`, `onEventDelete` callbacks; the Cortex adapter translates them to the commands below.
>
> | Keyboard action | DayFlow callback | IPC command path |
> |-|-|-|
> | Arrow keys (week/day grid navigation) | internal focus move | no IPC — view state only |
> | `Enter` / `Space` on focused event | `onEventClick` | no IPC — opens `onEventSelect` detail panel |
> | `Escape` | Cortex surface `onKeyDown` handler | no IPC — closes open detail / deselects |
> | `Delete` / `Backspace` on focused event | `onEventDelete` | `calendar_delete_event(eventId)` (E25 guard applies) |
> | Drag-and-drop via keyboard (plugin) | `onEventUpdate` | `calendar_reschedule_event(eventId, start, end, allDay)` |
>
> **[E26] Date/Time Interval Validation:**
>
> All calendar mutation commands validate `start < end` before persisting. If the invariant fails, the backend returns:
>
> ```json
> { "error": { "code": "INVALID_INPUT", "message": "event interval invalid: start must be before end" } }
> ```
>
> Affected commands: `calendar.rescheduleEvent`, `calendar.updateEvent`. Frontend never generates invalid intervals in normal operation; the guard is a server-side correctness invariant (E26, FR-015).
>
> **[E27] Upgrade Compatibility Governance (ADR-0018 Performance + Upgrade Gate):**
>
> Any calendar contract change (field shape, validation semantics, or error envelope behavior) must complete the calendar compatibility checklist in `docs/VERSIONING.md` and `docs/CODEGEN.md` before merge.
>
> Required paired validation evidence:
> - Frontend: `npm run test:dayflow-guardrails`
> - Backend: `cargo test -p cortex-storage` and `cargo test -p cortex-app`
> - Linked paired PRs in `cortex-os-frontend` and `cortex-os-backend`

> **[E28-BugFix] Scheduling UX Consolidation (final state):**
>
> Two drag/drop bugs (stale callback closure and fake `DragEvent` crash path) drove a shift to a modal-first scheduling flow in Week view.
>
> **Final UX model in this branch:**
> - Unscheduled sidebar drag scheduling is removed from `WeekDashboard`.
> - Scheduling entrypoints are click-based:
>   - header `Schedule` button
>   - DayFlow toolbar `+` (intercepted via `onAddClick`)
> - Both open `ScheduleTaskModal` (task picker + time picker).
>
> **Calendar projection model (`useWeekDashboard.calendarItems`):**
> - `googleEvents`: `events.filter(e => e.source === 'google')`
> - `scheduledTaskItems`: derived from `tasks` where `startTime/endTime` are set
> - `calendarItems = [...googleEvents, ...scheduledTaskItems]`
> - De-dup guard excludes task items already represented by a Google bridge event (`taskEventIds`).
>
> **Authoritative scheduling/mutation paths:**
>
> | Action | Frontend entry point | IPC / persistence path |
> |-|-|-|
> | Schedule existing task from modal | `handleScheduleExistingTask` | `updateTask` + `calendar_schedule_task` |
> | Schedule new task from modal | `handleScheduleNewTask` | `addTask` (`vault_create_page(kind:"task")` with `start_time/end_time`) |
> | Move/resize scheduled task | `handleDayflowEventUpdate` / `persistResizedItem` | `updateTask` (+ `calendar_reschedule_event` for Google-linked bridge events) |
> | Move pure calendar event | `handleDayflowEventUpdate` | `updateCalendarEvent` / `calendar_reschedule_event` |
>
> **Stale-closure guard retained:**
> `tasksRef` and `eventsRef` are live refs used by DayFlow callbacks so handler logic always reads current state.
>
> **External-drop callback note:**
> `handleExternalDrop` and DayFlow native window drop listeners remain implemented, but `WeekDashboard` no longer wires `onExternalDrop` in this branch's final UI flow.

> **[E26] Responsive Breakpoints (frontend-only, no IPC impact):**
>
> `DayflowCalendarSurface` wraps `DayFlowCalendar` in a responsive container. No command signatures change. Breakpoint behaviour is a CSS/layout concern only.
>
> | Breakpoint | Minimum width | Calendar behaviour |
> |-|-|-|
> | Mobile (sm) | `< 640px` | Day view only; week/month columns collapse to single-day scroll |
> | Tablet (md) | `640px – 1023px` | Week view with compressed column widths |
> | Desktop (lg+) | `≥ 1024px` | Full week/month grid at design spec widths |

### Integrations (Google Calendar + Obsidian Linked Vault)

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `integrations.googleAuth` | Authenticate with Google | — | `string` (auth URL) | Settings Integrations | `integrations_google_auth` (requires `GOOGLE_CLIENT_ID` + `GOOGLE_CLIENT_SECRET`) |
| `integrations.googleCalendars` | List available calendars with metadata | — | `GoogleCalendarMetadata[]` (`id`, `summary`, `backgroundColor?`, `primary`) | Settings Integrations, Week planner filtering | `integrations_google_calendars` |
| `integrations.triggerSync` | Trigger two-way sync manually and emit progress telemetry | — | `boolean` | Settings Integrations, background sync queue | `integrations_trigger_sync` |
| `integrations.deleteMirroredEvent` | Delete mirrored Google source event and linked local mirror entities | `pageId` | `boolean` | Week planner mirrored-delete flow | `integrations_delete_mirrored_event` |
| `obsidian.linkAdd` | Register an external Obsidian vault link | `request: { root_path, mode: "read_only"\|"read_write", include_paths?, exclude_paths? }` | `VaultLink` | Settings Integrations (Linked Obsidian Vault panel) | `obsidian_link_add` |
| `obsidian.linkList` | List linked Obsidian vaults | — | `VaultLink[]` | Settings Integrations | `obsidian_link_list` |
| `obsidian.linkRemove` | Remove a linked Obsidian vault | `request: { link_id }` | `void` | Settings Integrations | `obsidian_link_remove` |
| `obsidian.linkSetMode` | Set linked vault mode | `request: { link_id, mode: "read_only"\|"read_write" }` | `VaultLink` | Settings Integrations | `obsidian_link_set_mode` |
| `obsidian.syncNow` | Trigger immediate linked-vault sync run | `request: { link_id }` | `SyncRun` | Settings Integrations sync controls | `obsidian_sync_now` |
| `obsidian.syncStatus` | Read linked-vault sync status and recent jobs | `request: { link_id, limit? }` | `SyncStatus` | Settings Integrations status/progress panel | `obsidian_sync_status` |
| `obsidian.noteSave` | Save a linked Obsidian note with optimistic concurrency and conflict return | `request: { page_id, base_hash, markdown }` | `LinkedNoteSaveResult` | Vault Workbench (source editor) | `obsidian_note_save` |
| `obsidian.noteInspect` | Inspect linked-note source/sync/index metadata for the Vault Workbench inspector | `request: { page_id }` | `LinkedNoteInspectorStatus \| null` | RightDrawer note `Inspect` tab (linked notes only) | `obsidian_note_inspect` |

> **Integration settings payload updates:**
>
> `IntegrationSettings` now includes:
> - `syncedCalendars: string[]` (calendar IDs/names selected for sync)
> - `editableCalendars: string[]` (subset selected for writable task-mirror workflow)
> - `mirrorMigrationV1Done: boolean` (one-time migration marker)
>
> Existing fields (`googleCalendarConnected`, `googleCalendarEmail`, `syncEnabled`) are unchanged.
>
> **Obsidian linked-vault responses (ADR-0019):**
> - `VaultLink` includes: `linkId`, `provider`, `rootPath`, `mode`, `enabled`, `createdAt`, `updatedAt`, `lastScanAt?`, `lastError?`.
> - `SyncRun` includes: `runId`, `linkId`, `status`, `startedAt`, `finishedAt?`, `phase`, `processed`, `total`, `error?`.
> - `SyncStatus` includes: `link`, `activeRun?`, `recentRuns[]`, `queueCounts`.
> - `LinkedNoteSaveResult` uses status-tagged union:
>   - `saved`: `{ status: "saved", note, sourceHash }`
>   - `conflict`: `{ status: "conflict", serverMarkdown, serverHash, message }`
>
> **Linked note save conflict policy (read_write mode):**
> - backend compares `request.base_hash` with current source-file hash.
> - mismatch does not throw; response returns `status: "conflict"` with server payload.
> - UI must present an explicit merge decision path (`use server` / `overwrite with my changes`).
>
> **Linked note inspector metadata (`obsidian.noteInspect`):**
> - Returns `null` when the page is not a linked Obsidian note.
> - `LinkedNoteInspectorStatus` includes link mode/root path + note source path/hash, plus optional `manifest`, `indexState`, and `syncSummary` objects for Vault Workbench diagnostics.

### Search

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `search.global` | Search all content | `query`, `limit?` | `SearchResult[]` | CommandPalette, AI agent `searchBrain` | `SearchService::global` |
| `search.semantic` | Vector-only semantic search | `query`, `limit?` | `SearchResult[]` | Future semantic tab / agent tools | `search_semantic` |
| `search.graphNeighbors` | Traverse related pages by link distance | `pageId`, `depth?`, `limit?` | `SearchResult[]` | CommandPalette related graph badges | `search_graph_neighbors` |
| `search.graphLinks` | Exact directed link inspection (incoming/outgoing/both) | `pageId`, `direction?`, `limit?` | `GraphLinkResult[]` | RightDrawer note inspector backlinks/outgoing diagnostics | `search_graph_links` |
| `search.graphSuggestLinks` | Suggest unlinked related pages | `pageId`, `limit?` | `SearchResult[]` | Link suggestion workflows (Phase 4 prep) | `search_graph_suggest_links` |

> **`GraphLinkResult` response shape (summary):**
> `pageId`, `title`, `path`, `resultType`, `direction`, `relation`, `weight`, `sourcePageId`, `targetPageId`.
> Use `search.graphLinks` for exact backlinks/outgoing links; use `search.graphNeighbors` for traversal/ranking UX.

## Phase 3 Page-Centric Notes

Search commands are powered by:

- FTS5 lexical ranking (`pages_fts`)
- sqlite-vec chunk vectors (`search_chunk_vec`)
- bidirectional wikilink graph edges (`graph_edges`)

Embedding provider options:

- `same_as_model` (default) resolves to the active model provider family where embedding support exists
- explicit providers: `openai | gemini | ollama | hash`
- fallback policy: if resolved provider embeddings are unavailable, backend falls back to `hash` and surfaces diagnostic status

### Quick Capture

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `capture.save` | Save quick capture entry | `text` (canonical), `content?` (legacy compatibility) | `Page` | TodayDashboard capture widget | `capture_save` (appends to `Quick Capture/YYYY-MM-DD.md`) |

### AI (Phase 4 — backend IPC foundation)

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `ai_get_models` | List available models | — | `AIModel[]` | Settings model picker, RightDrawer model selector | `ai_get_models` |
| `ai_chat` | Send chat message and emit stream events | `request: { history, message, model_id?, enable_agent? }` | `ChatResponse { text, tool_results[], request_id }` | RightDrawer AI panel | `ai_chat` |
| `ai_summarize` | Summarize note content | `request: { title, body }` | `string` | Note summary panel | `ai_summarize` |
| `ai_generate_image` | Generate image artifact | `request: { prompt }` | `string` (data URI) | Project artifact generation | `ai_generate_image` |
| `ai_transcribe` | Transcribe audio using selected STT provider | `request: { audio_base64, tier?, mime_type? }` | `string` | Voice input transcription | `ai_transcribe` (`sttProvider`: `local_whisper\|openai\|gemini`, current default `gemini`) |
| `ai_synthesize` | Text to speech using selected TTS provider | `request: { text, voice? }` | `string` (base64 WAV) | Chat auto-speak | `ai_synthesize` (`ttsProvider`: `gemini\|openai\|local`) |
| `ai_validate_key` | Validate provider credential/endpoint format | `request: { provider, key }` | `boolean` | Settings provider verification | `ai_validate_key` |
| `ai_rag_query` | Retrieval-augmented query over vault/search index | `request: { query, limit? }` | `AIRagResult { request_id, answer, context[] }` | AI assistant context mode | `ai_rag_query` |
| `ai_suggest_links` | Suggest related pages from embeddings+graph | `request: { page_id, limit? }` | `Page[]` | Note linking workflows | `ai_suggest_links` |
| `review_list` | List Morning Review items | `status?` (`pending|approved|rejected`) | `ReviewQueueItem[]` | Morning Review UI | `review_list` |
| `review_approve` | Approve review item | `itemId`, `editedJson?` | `boolean` | Morning Review UI | `review_approve` |
| `review_reject` | Reject review item | `itemId` | `boolean` | Morning Review UI | `review_reject` |
| `token_usage` | Query token/cost usage rollups | `days?` | `TokenUsageEntry[]` | Settings usage dashboard | `token_usage` |

> **AI settings extension (ADR-0019):**
> - `AISettings.embeddingProvider`: `same_as_model | openai | gemini | ollama | hash`
> - `settings_get` / `settings_update` must preserve this field for frontend roundtrip behavior.

### Secret Storage (Phase 4)

| Command | Description | Request fields | Response fields | Frontend usage | Backend handler |
|---------|-------------|----------------|----------------|----------------|----------------|
| `secret_set` | Store encrypted secret envelope | `request: { secret_key, plaintext }` | `boolean` | Secure key rotation flows | `secret_set` |
| `secret_get` | Retrieve decrypted secret value | `secretKey` | `string \| null` | Secure settings bootstrap | `secret_get` |
| `secret_delete` | Remove encrypted secret | `secretKey` | `boolean` | Credential revoke flows | `secret_delete` |

## Event Streams

Realtime frontend invalidation for Phase 3 uses Tauri event channels in addition to request/response commands.

| Event | Description | Payload fields | Frontend usage | Backend emitter |
|-------|-------------|----------------|----------------|-----------------|
| `page_created` | A page/entity was created | `pageId`, `kind`, `updatedAt?` | `stores/realtimeStore.ts` subscription; triggers view invalidation/remount in `App.tsx` | Page-creation handlers (`vault_create_page`, `capture_save`, etc.) |
| `page_updated` | A page/entity was updated | `pageId`, `kind`, `updatedAt?` | `stores/realtimeStore.ts` subscription; triggers view invalidation/remount in `App.tsx` | Page-update handlers (`page_update_props`, `page_update_body`, `habits_toggle`) |
| `page_deleted` | A page/entity was deleted | `pageId`, `kind`, `updatedAt?` | `stores/realtimeStore.ts` subscription; triggers view invalidation/remount in `App.tsx` | `vault_delete` |
| `integrations_sync_progress` | Google Calendar sync lifecycle/progress updates | `phase`, `calendarId?`, `calendarName?`, `processed?`, `total?`, `message?` | Background sync status UX (WeekDashboard + Settings) | `integrations_trigger_sync` |
| `obsidian_sync_progress` | Linked Obsidian vault sync lifecycle/progress updates | `phase`, `linkId`, `processed?`, `total?`, `message?`, `runId?`, `status?` | Settings linked-vault progress panel | linked-vault runtime + `obsidian_sync_now` |
| `ai_stream_chunk` | Streamed AI text chunk | `requestId`, `text` | `services/aiService.ts` streaming updates in RightDrawer | `ai_chat` |
| `ai_stream_done` | Stream completion metadata | `requestId`, `inputTokens`, `outputTokens`, `estimatedCostUsd` | `services/aiService.ts` completion handling | `ai_chat` |
| `ai_stream_error` | Stream failure notification | `requestId`, `error`, `provider`, `code` | `services/aiService.ts` error/retry UX | `ai_chat` provider adapter failure path |

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
