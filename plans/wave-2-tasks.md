# Wave 2: Tasks

**Status:** done (2026-10-07); D4–D8 folded into PLAN · **PLAN:** §0 (Overdue, "Today", Subtasks), §1, §2, §4, §6 (dashboard and detail), §9 phase 2

## Goal

A logged-in user can create, view, edit, and delete their own tasks, change a task's status from the list, and filter, sort, and page through the list. Tasks form a tree up to 5 levels deep: the user adds subtasks by hand at any level below the limit, and deleting a task deletes its whole subtree. A nightly job marks late tasks `OVERDUE`. Every task query is scoped to the authenticated user, and another user's task is always a `404`.

## Out of scope

- Every AI feature (Wave 3). The task form and the subtask section keep empty slots for them.
- Moving a task to another parent, and reordering existing subtasks (PLAN §9, Later).
- Deriving a parent's status from its subtasks (PLAN §2).
- Bulk actions (multi-select, bulk delete, bulk status).
- Showing the tree inline on the dashboard. The dashboard lists top-level tasks; the tree is explored through the detail page.

## Decisions for this wave

Confirmed by the maintainer on 2026-10-07. D1 restates decisions already in PLAN. D2–D8 are new, and D4–D6 change PLAN wording, so the orchestrator folds them into PLAN after the wave. Details below the table.

| # | Decision | Outcome |
|---|---|---|
| D1 | Task tree | As PLAN §0 and §1: one `Task` entity with a self-referencing `PARENT_TASK_ID` (`ON DELETE CASCADE`), max depth `app.tasks.max-depth` (default `5`), parent fixed at creation. One shared depth helper, built on a recursive-CTE ancestors query. See below. |
| D2 | `PATCH` semantics | Absent field = unchanged; `null` = clear. Only `dueDate` and `complexity` can be cleared. No new dependency. See below. |
| D3 | Lookup order and caching | The lookups are loaded once at startup and cached, ordered by `ID` (seed order). Sorting by `priority` uses that order (`LOW < MEDIUM < HIGH`), not the name, so `priority,desc` puts `HIGH` first. |
| D4 | Overdue on write | The overdue rule also runs on every write, not only in the nightly job. See below. |
| D5 | Subtask order | A new `TASK.POSITION` column keeps the order subtasks were created in, including the order of accepted AI drafts. See below. |
| D6 | UI edits use `PATCH` | The edit form sends `PATCH` with only the changed fields. `PUT` stays in the API for API clients. See below. |
| D7 | Stable list order | Every sort gets the tie-breakers `createdAt,desc` then `id,asc`, so pages never repeat or skip a task. `dueDate` sorts nulls last in both directions. |
| D8 | Text search | `q` is trimmed; blank means no filter. At most 100 characters (else `400 VALIDATION_ERROR`). Matching is a case-insensitive `LIKE '%q%'` on title and description, with `%`, `_`, and `\` escaped. No full-text index in v1. |

### D1: task tree

- **Ancestors:** `TaskRepository` has a native recursive CTE that walks `PARENT_TASK_ID` upward and returns `(id, title)` rows, root first. It's also filtered by `USER_ID`, as a second guard.
- **Depth helper** (`TaskService`, reused by 2.4 and Wave 3's breakdown): `depth = ancestors.size() + 1`; `canAddSubtasks = depth < maxDepth`. One query per call, at most 4 rows, so no caching.
- **Subtask create:** the new task takes the parent's `user`. Because a new task has no descendants, setting its parent can't create a cycle, so there is no cycle check.
- **Delete:** one `DELETE` of the task row. The FK cascade removes the subtree in the same statement. No tree walk in Java, and JPA must not try to cascade on its own (no `@OneToMany(cascade = REMOVE)` on `children`).

### D2: `PATCH` semantics

PLAN §4 says `PATCH` takes "any subset of the `PUT` fields". To clear a due date the client sends `"dueDate": null`, so the API must tell "absent" from "null".

- **No new dependency.** `jackson-databind-nullable` isn't on the classpath, the `pom.xml` hotspot rule allows no new dependencies in this wave, and it may not support Jackson 3 (Boot 4).
- **Approach:** the worker picks one and proves it with tests. Suggested, in order:
  1. `PatchTaskRequest` as a record of `Optional<T>` fields, if Boot 4's Jackson maps an absent field to `null` and an explicit `null` to `Optional.empty()`. Validate with container element constraints (`Optional<@Size(max = 100) String>`).
  2. Otherwise, a class whose setters record which fields were present.
- **Rules:**
  - `dueDate: null` clears the date; `complexity: null` clears the complexity.
  - `title`, `description`, `priority`, or `status` sent as `null` → `400 VALIDATION_ERROR` with an `errors` entry for that field.
  - An empty body `{}` is valid and changes nothing except `updatedAt`.
  - Present fields get the same validation as `PUT`.

### D4: overdue on write

PLAN §2 only resets an `OVERDUE` task back to `TODO` on write; it only sets `OVERDUE` in the nightly job. So a task created today with yesterday's due date would show `TODO` until 00:05. Proposed rule, applied by `TaskService` after every create, `PUT`, `PATCH`, and subtask create:

- If the resulting status is `TODO` or `IN_PROGRESS` and `dueDate < today` (from the `Clock`, in `app.timezone`) → `OVERDUE`.
- If the resulting status is `OVERDUE` and `dueDate` is today, later, or cleared → `TODO` (the existing PLAN rule).
- A user still can't *send* `OVERDUE` (`400 INVALID_STATUS`). Sending `TODO` or `IN_PROGRESS` on a task whose due date is still past just ends up `OVERDUE` again; the response shows it. `DONE` always sticks.

The nightly job stays: it catches tasks whose due date passes while nobody edits them.

### D5: subtask order

The breakdown flow (Wave 3) lets the user reorder drafts before accepting, and the batch is created in one request, so every subtask in it gets the same `createdAt`. Ordering by `createdAt` then `id` would scramble it.

- The task schema migration also adds `TASK.POSITION INT NOT NULL DEFAULT 0`.
- `POST /tasks/{id}/subtasks` gives new subtasks `max(sibling POSITION) + 1, + 2, …` in request order.
- `subtasks` in `GET /tasks/{id}` is ordered by `POSITION`, then `createdAt`, then `id`.
- `POSITION` is internal: it never appears in the API. Top-level tasks keep `0` and are ordered by the list's sort.
- Two concurrent subtask creates under one parent may produce equal positions. That's acceptable: the tie-breakers keep the order stable.

### D6: UI edits use `PATCH`

The task object the UI holds can have `status: OVERDUE`. A full `PUT` of the edited form would send it back and get `400 INVALID_STATUS`.

- The edit form diffs against the loaded task and sends `PATCH` with only the changed fields. A no-change save sends nothing and just leaves edit mode.
- The status control offers `TODO`, `IN_PROGRESS`, and `DONE`. On an `OVERDUE` task it also shows `OVERDUE` as the current, disabled option.
- The UI never decides overdue-ness itself: the overdue marker comes from `status === 'OVERDUE'`, never from comparing dates in the browser.
- Dates stay `YYYY-MM-DD` strings end to end (native `<input type="date">`). Never `new Date(dueDate)`, which shifts by time zone.

## Current state

After Wave 1:

- API: `TASK` exists from V1 but has no `DUE_DATE`, no title check, nullable `USER_ID`, `STATUS_ID`, and `PRIORITY_ID`, and the unused `TASK_SUBTASK` table. There are no task or lookup entities, repositories, services, or controllers. V2 is the email index; the task migration takes the next free number.
- API: `ErrorCode` already has every code this wave needs (`INVALID_STATUS`, `INVALID_PRIORITY`, `INVALID_COMPLEXITY`, `SUBTASK_DEPTH_EXCEEDED`, `TASK_NOT_FOUND`, `VALIDATION_ERROR`). `CurrentUserService` returns the caller's id. `AppProperties` has `timezone`, `jwt`, `ai`, and `cors`, but no `tasks`. The `Clock` bean is in `AppConfig`. `support/IntegrationTest` is the Testcontainers base class, and `JwtTestSupport` issues test tokens.
- API: timestamps follow the `AGENTS.md` rule (`Instant` from the `Clock`, stored as UTC `TIMESTAMP`).
- UI: `DashboardPage`, `NewTaskPage`, and `TaskDetailPage` are placeholders already routed under `ProtectedRoute`. There is no `tasks.json` namespace. `errors.json` still says `SUBTASK_DEPTH_EXCEEDED` means "a subtask can't have subtasks".
- UI: `apiRequest` in `src/api/client.ts` handles every method, JSON, `ApiError` (with `code` and field `errors`), and the 401 handler. `FormField` exists from 1.7. `src/test/renderRoute.tsx` takes `{ token, routes }`, and `src/test/fetchMock.ts` stubs `fetch`. There's no date library and no debounce helper, and none may be added (`package.json` hotspot rule).

## Tasks

API track: 2.1 → 2.2 → 2.3, 2.4, 2.5. UI track: 2.6 → 2.7, 2.8. The tasks after a fork may run in parallel (they own separate files), but with one worker per repo, run them in the order listed. The UI codes against the PLAN §2 and §4 contract plus D2–D8, not the running API, so it may start before Checkpoint A.

### 2.1 API: task schema, entities, and lookups

- **Repo:** API · **PLAN:** §1, §2 (Task object, Validation), §4 (`GET /lookups`)
- **Changes:**
  - A new migration with the next free version (e.g. `…__task-schema.sql`):
    - `DUE_DATE DATE NULL`; `CHECK (TITLE ~ '\S')`; `USER_ID`, `STATUS_ID`, `PRIORITY_ID` → `NOT NULL`
    - `PARENT_TASK_ID UUID NULL REFERENCES TASK(ID) ON DELETE CASCADE`, `CHECK (PARENT_TASK_ID <> ID)`, and an index on it (D1)
    - `POSITION INT NOT NULL DEFAULT 0` (D5)
    - drop `TASK_SUBTASK`
    - an index on `(USER_ID, STATUS_ID, DUE_DATE)`
  - `app.tasks.max-depth=5` in `application.properties`, bound as `AppProperties.Tasks(@Min(1) int maxDepth)`.
  - Entities: `Priority`, `TaskStatus`, `Complexity` (id + name), and `Task`: UUID id generated by the app, `title`, `description`, `dueDate` (`LocalDate`), lazy `@ManyToOne` to `User`, `Priority`, `TaskStatus`, `Complexity` (nullable), and `parent` (column `PARENT_TASK_ID`), `position`, and `createdAt`/`updatedAt` as `Instant` from the `Clock`. No JPA cascade to children (D1).
  - Repositories for each. `TaskRepository` gets `findByIdAndUserId` and the ancestors query (D1).
  - `service/LookupService`: loads and caches the three lookups at startup, ordered by `ID` (D3); resolves a name to its entity or throws `INVALID_PRIORITY`, `INVALID_STATUS`, or `INVALID_COMPLEXITY`.
  - `controller/LookupController`: `GET /api/v1/lookups` → `{ priorities, statuses, complexities }`, each a list of names in seed order.
- **Acceptance:**
  - Every migration applies, and the context starts with `ddl-auto=validate`.
  - These fail at the DB level: a blank title, a null `USER_ID`, and a task that is its own parent. `TASK_SUBTASK` no longer exists.
  - Deleting a top-level task with `JdbcTemplate` removes a 3-level subtree below it.
  - The ancestors query returns the path root first, and nothing for a top-level task.
  - `GET /lookups` returns the seeded names in seed order; without a token → 401.
  - Each unknown name produces its own error code.
- **Commit:** `feat(api): add task schema migration, task and lookup entities, lookups endpoint`

### 2.2 API: task CRUD

- **Repo:** API · **PLAN:** §2 (all), §4 (create, view, edit, status change, delete)
- **Changes:**
  - DTOs per PLAN §2: `CreateTaskRequest`, `UpdateTaskRequest` (`PUT`, every editable field), `PatchTaskRequest` (D2), `TaskResponse`, `AncestorSummary`, and `SubtaskSummary` (with `subtaskCount`). `ancestors`, `canAddSubtasks`, and `subtasks` are `@JsonInclude(NON_NULL)` and left `null` in list responses.
  - `service/TaskService`:
    - every load goes through `findByIdAndUserId(id, currentUser)`; a miss → `404 TASK_NOT_FOUND`, whether the task doesn't exist or belongs to someone else
    - defaults from PLAN §2: `priority = MEDIUM`, `status = TODO`, `complexity = null`
    - status rules from PLAN §2 plus D4: a user may not send `OVERDUE`; the overdue rule runs after every write
    - the depth helper (D1)
    - `subtasks` = direct children ordered per D5, with `subtaskCount` from one grouped count query (no N+1)
    - delete per D1
  - `controller/TaskController`: `POST` (201 + `Location`), `GET /{id}`, `PUT /{id}`, `PATCH /{id}`, `DELETE /{id}` (204). OpenAPI summaries and documented error responses.
- **Acceptance:** one integration test per §4 scenario except filter and subtask creation, plus:
  - user B gets 404 on user A's task for `GET`, `PUT`, `PATCH`, and `DELETE`
  - sending `OVERDUE` → 400 `INVALID_STATUS`; an unknown priority → 400 `INVALID_PRIORITY`
  - D4: creating a task due yesterday returns `OVERDUE`; moving an `OVERDUE` task's due date to today, or clearing it, returns `TODO`; marking it `DONE` sticks; `DONE` → `TODO` is allowed (fixed `Clock`)
  - D2: `PATCH {}` changes nothing but `updatedAt`; `PATCH {"dueDate": null}` clears the date; `PATCH {"title": null}` → 400; an absent field is untouched
  - deleting a task removes its children and grandchildren (rows inserted directly)
  - `GET /tasks/{id}` on a depth-3 task returns 2 ancestors root first, only its direct subtasks in `POSITION` order with correct `subtaskCount`s, and `canAddSubtasks: true`; on a depth-5 task, `canAddSubtasks: false`
- **Commit:** `feat(api): add task crud with ownership and status rules`

> **Checkpoint A:** `docker compose up --build task-manager-db task-manager-api`. In Swagger UI: create a task, fetch it, `PATCH` its status, clear its due date with `PATCH`, and delete it. A second user gets 404 on the first user's task.

### 2.3 API: list, filter, sort, and paginate

- **Repo:** API · **PLAN:** §4 (filter scenario, `GET /tasks` query parameters)
- **Owns:** `service/TaskQueryService`, `repository/TaskSpecifications`, `dto/PageResponse`, and `controller/TaskQueryController` (maps `GET /api/v1/tasks`). Don't edit `TaskController`.
- **Changes:**
  - JPA Specifications for every PLAN §4 filter, always `AND user = currentUser`.
  - `includeSubtasks=false` (default) → `parent IS NULL`; `true` → every depth.
  - Sort on `dueDate`, `priority` (by ID, D3), `createdAt`, and `title` only; any other field → 400 `VALIDATION_ERROR`. Tie-breakers and nulls per D7.
  - `size` default 20, capped at 100; `page` < 0 or `size` < 1 → 400.
  - `q` per D8.
  - Fetch the lookups through `LookupService`'s cache or a fetch join, so a page costs a fixed number of queries.
  - `PageResponse<T>` with PLAN §4's shape.
- **Acceptance:** tests for:
  - each filter alone and combined; repeated `status` values
  - date-range bounds being inclusive
  - `q` matching title and description in any letter case, and a `q` containing `%` matching literally
  - default sort, nulls last on `dueDate` both ways, `priority,desc` putting `HIGH` first
  - two tasks with equal sort keys keep a stable order across pages
  - `includeSubtasks` both ways
  - an invalid status or sort field → 400
  - results limited to the caller's own tasks
- **Commit:** `feat(api): add task filtering, sorting and pagination`

### 2.4 API: create subtasks

- **Repo:** API · **PLAN:** §0 (Subtasks), §4 (add subtask, `POST /tasks/{id}/subtasks`)
- **Owns:** `service/SubtaskService`, `controller/SubtaskController`, and `dto/CreateSubtaskRequest`.
- **Changes:**
  - Create 1–10 subtasks under a parent the caller owns, in one transaction, with the same defaults and validation as a task create.
  - Parent at the maximum depth (2.2's depth helper) → 400 `SUBTASK_DEPTH_EXCEEDED`.
  - Positions per D5, and the overdue rule per D4.
  - Return `201` with the created tasks, in request order.
- **Acceptance:** tests for:
  - success, with the new subtasks visible in `GET /tasks/{id}` in request order (D5), after an earlier batch
  - building a chain down to depth 5 through the endpoint succeeds
  - adding under a depth-5 task → 400 `SUBTASK_DEPTH_EXCEEDED`, and nothing is written
  - 0 or 11 items → 400; one invalid item → 400 and nothing is written
  - a parent owned by another user → 404
- **Commit:** `feat(api): add subtask creation endpoint`

### 2.5 API: overdue job

- **Repo:** API · **PLAN:** §2 (Status rules)
- **Owns:** `service/OverdueTaskScheduler` and `config/SchedulingConfig`.
- **Changes:**
  - Enable scheduling. Cron `0 5 0 * * *` in the `app.timezone` zone.
  - One bulk update: status `TODO` or `IN_PROGRESS` and `dueDate < today` (from the `Clock`, in `app.timezone`) → `OVERDUE`, with `updatedAt` refreshed. Status IDs resolved by name through `LookupService`.
  - Log how many tasks changed (a count only, no task data).
  - A public method the tests call directly.
- **Acceptance:** with a fixed `Clock`, TODO and IN_PROGRESS tasks with a past due date become OVERDUE, across users and at every depth. Unchanged: tasks due today, future tasks, DONE tasks, tasks with no due date, and tasks already OVERDUE (their `updatedAt` too).
- **Commit:** `feat(api): add scheduled overdue task job`

### 2.6 UI: task API layer and shared components

- **Repo:** UI · **PLAN:** §2 (Task object), §4 (endpoints)
- **Changes:**
  - `src/api/tasks.ts`: the types `Task`, `AncestorSummary`, `SubtaskSummary`, `PageResponse<T>`, `TaskFilters`, and `Lookups`, plus a function per task, subtask, and lookup endpoint. `patchTask` sends only the fields it's given, keeping explicit `null`s (D2).
  - `src/api/queries/tasks.ts`: TanStack Query hooks. Keys: `['tasks', filters]`, `['task', id]`, `['lookups']` (`staleTime: Infinity`). Mutations invalidate `['tasks']` and the affected `['task', id]` and its parent's.
  - Shared components: `PriorityBadge`, `StatusBadge`, `ComplexityBadge`, `ConfirmDialog`, `ErrorMessage` (an `ApiError.code` → `errors.json` text, via the existing `errorKey`).
  - A `useDebouncedValue` hook (no dependency).
  - `tasks.json` in both locales, registered in `src/i18n/index.ts`, `i18next.d.ts`, and `locales.test.ts`, like `auth.json` in 1.7.
  - Update `SUBTASK_DEPTH_EXCEEDED` in `errors.json` (both locales): the task is at the maximum depth.
- **Acceptance:** unit tests for query-string building (repeated `status`, dates, sort, page, `includeSubtasks`), for `patchTask` keeping `null` and dropping `undefined`, and for the badges rendering localized labels in both languages.
- **Commit:** `feat(ui): add task api layer, query hooks and shared task components`

### 2.7 UI: dashboard

- **Repo:** UI · **PLAN:** §4 (filter scenario, quick status change), §6
- **Owns:** `src/pages/DashboardPage.tsx` and `src/features/tasks/list/**`.
- **Changes:**
  - A task list (top-level tasks by default) showing title, status, priority, complexity, due date, and an overdue marker (`status === 'OVERDUE'`, D6). Each row links to `/tasks/:id`.
  - Filters: multi-select status, priority, and complexity; due-from and due-to dates; debounced text search; an "include subtasks" toggle. Options come from `['lookups']`.
  - Sort control and pagination.
  - Filters, sort, and page live in URL search params. Invalid params in the URL are dropped, not sent.
  - A quick done/undone toggle (`PATCH { status }`) with an optimistic update, rolled back on error. The row shows the status from the response, which may be `OVERDUE` (D4).
  - A "New task" link to `/tasks/new`.
  - Empty, loading, and error states.
- **Acceptance:** component tests:
  - changing a filter updates the URL and the request
  - the status toggle sends `PATCH` and rolls back on error
  - the empty state renders
- **Commit:** `feat(ui): add task dashboard with filters, sorting and pagination`

### 2.8 UI: task form, detail, and subtasks

- **Repo:** UI · **PLAN:** §4 (create, view, edit, delete, add subtask), §6
- **Owns:** `src/pages/NewTaskPage.tsx`, `src/pages/TaskDetailPage.tsx`, and `src/features/tasks/{form,detail,subtasks}/**`.
- **Changes:**
  - A `TaskForm` for create and edit. Client-side validation mirrors PLAN §2; server field errors map onto the fields. Edit sends `PATCH` with only the changed fields; the status control follows D6.
  - The detail page:
    - an ancestor breadcrumb above the title, each item linking to that task
    - every field, and the direct subtask list in API order, each linking to its own page and showing its `subtaskCount` when non-zero
    - an "add subtask" form, hidden when `canAddSubtasks` is `false`
    - edit, and delete with confirmation. After delete, go to the parent's page, or to `/` for a top-level task.
  - A 404 state for `TASK_NOT_FOUND`.
  - Empty slots for Wave 3: `<AiSuggestSlot />` next to the form and `<AiBreakdownSlot />` in the subtask section, each rendering nothing.
- **Acceptance:** component tests:
  - create submits the right payload
  - server validation errors show on the matching fields
  - edit sends only the changed fields, and an `OVERDUE` task saves without sending `status`
  - the breadcrumb renders the ancestors in order
  - the add-subtask form shows on a subtask with `canAddSubtasks: true` and is hidden when it's `false`
  - delete asks for confirmation, then navigates to the parent
- **Commit:** `feat(ui): add task create, detail, edit, delete and subtasks`

> **Checkpoint B (wave done):** run `docker compose up --build`. Create a task, add subtasks down to depth 5 (the add form disappears there), and follow the breadcrumb back up. Filter, sort, and page the dashboard, reload, and the filters survive. Toggle a task done from the list. Create a task due yesterday and see `OVERDUE`. Delete a mid-level task and its subtree is gone. Switch to PT-BR. Run `./mvnw test` in the API and `npm run lint && npm run build && npm test` in the UI. Then commit the submodule pointers in the root: `chore: bump submodules (wave 2)`.

## After this wave

- Fold D4 (overdue on write), D5 (`POSITION`, in PLAN §1), D6 (UI edits use `PATCH`), D7, and D8 into PLAN §1, §2, and §4.
- Mark this plan as done.
