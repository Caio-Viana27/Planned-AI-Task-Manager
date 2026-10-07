# AI Task Manager - Development Plan

This document describes the Behavior Driven Development (BDD) use cases, the RESTful API contract, and the delivery phases for the AI Task Manager. The backend is a monolithic Spring Boot 4.1.1 service (Java 25, Spring AI 2.0 with Google Gemini) on PostgreSQL 18. It uses stateless JWT authentication. The frontend is React 19 + TypeScript (see `AGENTS.md` for conventions).

## 0. Decisions

These decisions resolve ambiguities in earlier drafts. Change them here first if they change.

| Topic | Decision |
|---|---|
| Logout | **Stateless.** The client discards the token. Access tokens are short-lived (60 min). There are no refresh tokens and no `/logout` endpoint in v1. |
| JWT | HS256, signed and verified with Spring Security's OAuth2 resource server (`NimbusJwtEncoder`/`NimbusJwtDecoder`). The secret comes from the `JWT_SECRET` env var. Claims: `sub` = user id, `email`, `roles`, `exp`. |
| Password reset | **Specified here but delivered in a later phase** (Phase 5). It needs SMTP. Mailpit is used locally. |
| Overdue | `OVERDUE` stays a **stored status**. A scheduled job sets it (see §2 "Status rules"). |
| "Today" | Comes from the configurable app time zone `app.timezone` (default `America/Sao_Paulo`). `dueDate` is a `DATE`, with no time component. |
| Subtasks | **A tree, at most 5 levels deep.** Every task has at most one parent and any number of subtasks, which can have subtasks of their own. "Parent" and "subtask" are roles, not types: there is one `Task` entity. A top-level task is depth 1, and a task at the maximum depth (`app.tasks.max-depth`, default `5`) can't get subtasks. Deleting a task deletes its whole subtree. The parent is set only when a subtask is created and never changes, so the tree can't have cycles. Subtasks are hidden from the main list by default. They can be created manually or from AI drafts. |
| AI suggestions | "Enhance" and "Estimate" are **merged** into one `POST /api/v1/ai/suggest` endpoint. It takes a draft `{title, description}`, so it works on unsaved tasks too. Breakdown stays task-based. |
| Chat | **Query-only and stateless** in v1. The client may send the last few messages as context, and nothing is persisted. Creating and editing tasks through chat (tool calling) comes in a later phase. |
| Roles | `USER` and `ADMIN` are seeded. Every new account gets `USER`. There are no admin endpoints in v1. Every user sees only their own tasks. |
| User name | Sign-up takes `name`, a display name that isn't unique and maps to `USERS.NAME`. Users sign in with `email`. |

## 1. Task schema changes (new migration)

- `TASK.DUE_DATE DATE NULL`.
- A not-blank check on title: `CHECK (TITLE ~ '\S')`.
- `TASK.USER_ID` becomes `NOT NULL`, because every task belongs to a user.
- `TASK.STATUS_ID` and `TASK.PRIORITY_ID` become `NOT NULL`. The service applies defaults (see §2).
- `TASK.PARENT_TASK_ID UUID NULL REFERENCES TASK(ID) ON DELETE CASCADE`, with `CHECK (PARENT_TASK_ID <> ID)` and an index on `PARENT_TASK_ID`. A column holds exactly one parent per task, and the cascade deletes a whole subtree at any depth.
- Drop `TASK_SUBTASK`. It's empty, because no endpoint wrote tasks before this migration. V1 itself isn't edited.
- An index on `TASK (USER_ID, STATUS_ID, DUE_DATE)` for listing and filtering.
- `TASK.POSITION INT NOT NULL DEFAULT 0`: the order of a task among its siblings, in creation order (request order for a batch, including accepted AI drafts). It's internal and never appears in the API. Top-level tasks keep `0`.

The service layer enforces the depth limit (§0) and creates every subtask with its parent's `USER_ID`. The database doesn't check depth.

## 2. Shared API contract

### Task object

```json
{
  "id": "uuid",
  "title": "string (1-100)",
  "description": "string (1-500)",
  "dueDate": "2026-10-31 | null",
  "priority": "LOW | MEDIUM | HIGH",
  "status": "TODO | IN_PROGRESS | OVERDUE | DONE",
  "complexity": "EASY | MEDIUM | HARD | null",
  "parentTaskId": "uuid | null",
  "ancestors": [ { "id": "uuid", "title": "string" } ],
  "canAddSubtasks": true,
  "subtasks": [ "SubtaskSummary" ],
  "createdAt": "ISO-8601",
  "updatedAt": "ISO-8601"
}
```

`ancestors`, `canAddSubtasks`, and `subtasks` appear only in the single-task response (`GET /tasks/{id}`). List responses leave them out.

- `ancestors`: the path from the top-level task down to the direct parent, root first, for a breadcrumb. Empty for a top-level task. The task's depth is `ancestors.length + 1`.
- `canAddSubtasks`: `false` when the task is at the maximum depth. The UI uses it to hide "add subtask" and "Break down with AI", so it never needs to know the limit.
- `subtasks`: **direct** children only, never the whole subtree. To go deeper, open a subtask.

### SubtaskSummary object

```json
{
  "id": "uuid",
  "title": "string (1-100)",
  "status": "TODO | IN_PROGRESS | OVERDUE | DONE",
  "priority": "LOW | MEDIUM | HIGH",
  "dueDate": "2026-10-31 | null",
  "subtaskCount": 0
}
```

`SubtaskSummary` is a deliberately trimmed view, holding only what the subtask list on the parent's detail page displays. `subtaskCount` is the number of its direct children, so the list can show that a subtask has subtasks of its own. `description`, `complexity`, `parentTaskId`, and the timestamps are left out to keep `GET /tasks/{id}` small. To get a subtask's full details, call `GET /tasks/{subtaskId}`.

### Validation and defaults

- `title`: required, not blank, at most 100 characters. `description`: required, not blank, at most 500 characters.
- `priority` defaults to `MEDIUM`. `status` defaults to `TODO`. `complexity` defaults to `null`.
- Enum values are the lookup-table **names**. The service resolves names to IDs, and IDs never appear in the API. An unknown name returns `400`.

### Status rules

- A user may set `TODO`, `IN_PROGRESS`, or `DONE`, and may move between them freely (e.g. reopen a `DONE` task).
- Only the system sets `OVERDUE`. If a user sends `OVERDUE`, the API returns `400 INVALID_STATUS`.
- The overdue rule runs after every write (create, `PUT`, `PATCH`, subtask create), with "today" from `app.timezone`:
  - `TODO` or `IN_PROGRESS` with `dueDate < today` becomes `OVERDUE`. Sending `TODO` or `IN_PROGRESS` on a task that is still late ends up `OVERDUE` again, and the response shows it.
  - `OVERDUE` with a `dueDate` of today or later, or none, goes back to `TODO`.
  - `DONE` always sticks. Marking an `OVERDUE` task `DONE` is allowed.
- A scheduled job (`@Scheduled`, daily at 00:05 in `app.timezone`) applies the same rule in bulk, so tasks whose due date passes while nobody edits them become `OVERDUE`.
- A parent's status is **not** changed automatically by its subtasks in v1.

### Errors

Every error uses RFC 9457 `ProblemDetail`, plus a `code` property in English and an `errors` list for validation failures. The UI maps `code` to an i18n key.

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Bean Validation failed or a filter value is malformed |
| 400 | `INVALID_STATUS` / `INVALID_PRIORITY` / `INVALID_COMPLEXITY` | Unknown enum name, or `OVERDUE` set by a user |
| 400 | `SUBTASK_DEPTH_EXCEEDED` | Adding subtasks to (or breaking down) a task at the maximum depth (§0) |
| 401 | `UNAUTHORIZED` | Missing, invalid, or expired token |
| 401 | `BAD_CREDENTIALS` | Wrong email or password at sign-in |
| 404 | `TASK_NOT_FOUND` | The task doesn't exist **or belongs to another user**. The API never reveals that it exists. |
| 409 | `EMAIL_ALREADY_USED` | Sign-up with an existing email |
| 422 | `AI_INVALID_RESPONSE` | The model's output can't be parsed or validated |
| 429 | `AI_RATE_LIMITED` | The per-user AI quota is exceeded |
| 503 | `AI_UNAVAILABLE` | Gemini timed out, rate-limited us, or is offline |
| 405 | `METHOD_NOT_ALLOWED` | The endpoint doesn't support the HTTP method |
| 415 | `UNSUPPORTED_MEDIA_TYPE` | The request body isn't JSON |
| 500 | `INTERNAL_ERROR` | An unexpected server error. The response never includes the cause. |

Errors that Spring MVC raises itself get a `code` from their status: 400 → `VALIDATION_ERROR` (e.g. a missing required parameter), 405, 415, and 500 as above.

## 3. Authentication features

### BDD use cases

* **Sign up**
  * **Given** an unregistered user is on the registration page,
  * **When** they provide a valid email, name, and password (at least 8 characters),
  * **Then** an account with role `USER` is created, the password is hashed with BCrypt, and a JWT is returned for immediate login.
  * *Negative:* if the email already exists, the API returns `409 EMAIL_ALREADY_USED`. Invalid fields return `400 VALIDATION_ERROR`.

* **Sign in**
  * **Given** a registered user is on the login page,
  * **When** they provide their correct email and password,
  * **Then** the system returns a signed JWT that is valid for 60 minutes.
  * *Negative:* if the email is unknown or the password is wrong, the API returns `401 BAD_CREDENTIALS`. The same response is used for both cases.

* **Logout**
  * **Given** an authenticated user,
  * **When** they click "Log out",
  * **Then** the UI deletes the stored token, clears the TanStack Query cache, and redirects to the login page. No API call is made.

* **Expired session**
  * **Given** a user's token has expired,
  * **When** the UI receives a `401 UNAUTHORIZED`,
  * **Then** the UI clears the token and redirects to the login page with a "session expired" message.

* **Reset password** *(Phase 5)*
  * **Given** a user has forgotten their password,
  * **When** they submit their email in the "forgot password" flow,
  * **Then** the API always returns `200` (whether or not the email exists). If the email exists, the system emails a single-use link that expires in 30 minutes.
  * **And** when they submit a new password with a valid token, the password is updated and the token is consumed. If the token is expired or already used, the API returns `400`.

### Field limits

Sign-up fields (anything else is `400 VALIDATION_ERROR`):

| Field | Rule |
|---|---|
| `email` | Required, a valid email, at most 100 characters. |
| `name` | Required, not only spaces, at most 100 characters. Leading and trailing spaces are stripped before saving. |
| `password` | Required, at least 8 characters and at most 72 **UTF-8 bytes** (BCrypt only uses the first 72 bytes). Spaces are allowed and kept. No other complexity rules. |

Sign-in only requires `email` and `password` to be non-blank. It checks no formats or lengths, so a failed sign-in never reveals the sign-up rules. Anything that doesn't match, including a password over 72 bytes, is `401 BAD_CREDENTIALS`.

### Email case

Emails are case-insensitive. The API stores them lowercase (`Locale.ROOT`) and lowercases them on sign-in, and `/users/me` and `AuthResponse.user` return the stored lowercase email. The database enforces it too, with a unique index on `LOWER(EMAIL)`, so no write path can create two accounts that differ only in case. A concurrent sign-up that hits either email constraint returns `409 EMAIL_ALREADY_USED`, not `500`.

### Endpoints

| Method | Endpoint | Description | Request | Response |
|---|---|---|---|---|
| `POST` | `/api/v1/auth/signup` | Register | `{ email, name, password }` | `201` `{ token, expiresAt, user: { id, name, email } }` |
| `POST` | `/api/v1/auth/signin` | Authenticate | `{ email, password }` | `200` `{ token, expiresAt, user: { id, name, email } }` |
| `GET` | `/api/v1/users/me` | Current user profile | Bearer token | `200` `{ id, name, email, roles }` |
| `POST` | `/api/v1/auth/password-reset` | *(Phase 5)* Request a reset | `{ email }` | `200` (always) |
| `PUT` | `/api/v1/auth/password-reset` | *(Phase 5)* Confirm a reset | `{ token, newPassword }` | `204` |

Phase 5 adds a `PASSWORD_RESET_TOKEN` table (`USER_ID`, `TOKEN_HASH`, `EXPIRES_AT`, `USED_AT`). Only a hash of the token is stored. Reset requests are rate-limited per email and per IP.

## 4. Core task management

### BDD use cases

* **Create a task**
  * **Given** an authenticated user is on the dashboard,
  * **When** they submit a title, description, and optionally a due date, priority, and complexity,
  * **Then** the task is saved with `status = TODO` and appears in their task list.

* **View a task**
  * **Given** an authenticated user has tasks,
  * **When** they open one of them,
  * **Then** they see its full details, its direct subtasks, and the path of its ancestors.
  * *Negative:* opening another user's task returns `404 TASK_NOT_FOUND`.

* **Edit a task**
  * **Given** an authenticated user is viewing one of their tasks,
  * **When** they change any field and save,
  * **Then** the changes are saved, `updatedAt` is refreshed, and the status rules in §2 are applied.

* **Change status quickly**
  * **Given** a task in the list,
  * **When** the user changes only its status (e.g. ticks "done"),
  * **Then** only the status is updated (`PATCH`).

* **Filter, sort, and paginate tasks**
  * **Given** an authenticated user has many tasks,
  * **When** they filter by status (one or more), priority, complexity, due-date range, or text, and choose a sort order,
  * **Then** the list shows one page of matching top-level tasks.
  * *Negative:* an invalid filter value (e.g. `status=OPEN`) returns `400`.

* **Delete a task**
  * **Given** an authenticated user no longer needs a task,
  * **When** they confirm the deletion,
  * **Then** the task **and its whole subtree, at every depth,** are deleted in one transaction.

* **Add a subtask manually**
  * **Given** a task below the maximum depth (top-level or itself a subtask),
  * **When** the user adds a subtask,
  * **Then** the subtask is created one level below it and linked to it.
  * *Negative:* if the task is at the maximum depth, the API returns `400 SUBTASK_DEPTH_EXCEEDED`.

### Endpoints

| Method | Endpoint | Description | Request | Response |
|---|---|---|---|---|
| `POST` | `/api/v1/tasks` | Create a task | `{ title, description, dueDate?, priority?, complexity? }` | `201` Task, with a `Location` header |
| `GET` | `/api/v1/tasks/{id}` | Get one task | – | `200` Task (with `ancestors`, `canAddSubtasks`, and `subtasks`) |
| `PUT` | `/api/v1/tasks/{id}` | Replace editable fields | `{ title, description, dueDate, priority, status, complexity }` | `200` Task |
| `PATCH` | `/api/v1/tasks/{id}` | Partial update | Any subset of the `PUT` fields | `200` Task |
| `DELETE` | `/api/v1/tasks/{id}` | Delete the task and its whole subtree | – | `204` |
| `GET` | `/api/v1/tasks` | List and filter | See the query parameters below | `200` Page of Task |
| `POST` | `/api/v1/tasks/{id}/subtasks` | Create subtasks (manual or accepted AI drafts) | `[{ title, description, priority?, complexity?, dueDate? }]` (1–10 items) | `201` `[Task]` |
| `GET` | `/api/v1/lookups` | Enum values for the UI | – | `200` `{ priorities, statuses, complexities }` |

**`GET /api/v1/tasks` query parameters:**

- `status`: repeatable, e.g. `status=TODO&status=OVERDUE`
- `priority`, `complexity`: repeatable
- `dueFrom`, `dueTo`: ISO dates, inclusive
- `q`: case-insensitive substring match on title and description. Trimmed; blank means no filter; at most 100 characters. `%`, `_`, and `\` match literally. No full-text index in v1.
- `includeSubtasks`: default `false`, which lists only top-level tasks (no parent). `true` lists tasks at every depth.
- `page`: default `0`
- `size`: default `20`, capped at `100`. `page` < 0 or `size` < 1 returns `400`.
- `sort`: default `dueDate,asc`. Allowed fields: `dueDate`, `priority`, `createdAt`, `title`; any other returns `400 VALIDATION_ERROR`. `priority` sorts by seed order (`LOW < MEDIUM < HIGH`), so `priority,desc` puts `HIGH` first. `dueDate` sorts tasks without a due date last in both directions. Every sort is followed by the tie-breakers `createdAt,desc` then `id,asc`, so pages never repeat or skip a task.

**Page response:** `{ content: [Task], page, size, totalElements, totalPages }`.

**Subtask order:** `POST /tasks/{id}/subtasks` adds the new subtasks after the existing ones, in request order. `subtasks` in `GET /tasks/{id}` is ordered by `POSITION` (§1), then `createdAt`, then `id`.

**UI edits use `PATCH`:** the edit form sends only the changed fields, so it never sends back an `OVERDUE` status. Its status control offers `TODO`, `IN_PROGRESS`, and `DONE`, and shows `OVERDUE` as the current, disabled option on an overdue task. The UI never decides overdue-ness itself: it reads `status === 'OVERDUE'`. Dates stay `YYYY-MM-DD` strings end to end.

## 5. AI features (Google Gemini via Spring AI)

### Shared rules for every AI endpoint

- All model access goes through one `AiAssistantService` (in `service/`). Controllers and other services never touch `ChatModel`/`ChatClient` directly.
- **Structured output:** responses are mapped to Java records with Spring AI's structured output converter. Enum fields are checked against the lookup names. If parsing or validation fails, the API returns `422 AI_INVALID_RESPONSE`.
- **Resilience:**
  - Timeout of 20 s.
  - One retry, only on transient errors.
  - Gemini timeouts, `429`s, and `5xx` errors return `503 AI_UNAVAILABLE`.
  - The UI shows a localized "AI is temporarily unavailable" message and keeps the user's work intact.
- **Quota:** 30 AI requests per user per hour (configurable). Above that, the API returns `429 AI_RATE_LIMITED`.
- **Prompt hygiene:**
  - User and task text go in clearly delimited sections and are treated as data, never as instructions.
  - Only the requesting user's own tasks are ever placed in a prompt.
- **Context:** every prompt includes today's date and `app.timezone`, plus the user's locale (`Accept-Language`, `en` or `pt-BR`). Generated text is written in that language.
- **Accepting suggestions:** AI endpoints never write to the database. The user saves accepted suggestions through the normal task endpoints.
- **Config:** `GEMINI_API_KEY` and `GEMINI_MODEL` go in `.env` and are listed in `.env.example`.

### BDD use cases

* **Suggest improvements (enhance + estimate)**
  * **Given** a user is drafting or editing a task,
  * **When** they click "Suggest with AI",
  * **Then** the AI returns a rewritten title and description, a suggested priority and complexity, and a short reasoning.
  * **And** the UI shows the current and suggested values side by side. The user can accept them all, pick fields one by one, or dismiss them.
  * **And** for a saved task, accepting calls `PATCH /tasks/{id}`. For a draft, accepting fills the create form.

* **Break down into subtasks**
  * **Given** a saved task below the maximum depth,
  * **When** the user asks the AI to break it down,
  * **Then** the AI returns 2–8 draft subtasks, each with a title, description, priority, and complexity.
  * **And** the prompt includes the titles of the task's ancestors, root first, so drafts for a deep subtask fit its context.
  * **And** the user can edit, remove, or reorder the drafts, then accept. Accepting calls `POST /tasks/{id}/subtasks` with the remaining drafts.
  * *Negative:* if the task is at the maximum depth, the API returns `400 SUBTASK_DEPTH_EXCEEDED`.

* **Chat assistant (query-only)**
  * **Given** an authenticated user opens the chat panel,
  * **When** they ask something like "Do I have overdue tasks?",
  * **Then** the backend loads up to 20 of the user's tasks that aren't `DONE` and puts them in the prompt. Tasks are ordered by `OVERDUE` first, then `dueDate` ascending (nulls last), then priority descending. The AI's reply is based only on those tasks.
  * **And** if the question is about something the assistant can't do in v1 (e.g. "create a task"), it explains how to do it in the UI.
  * **And** if Gemini is unavailable, the chat shows the localized unavailable message and the user's input stays in the box.

### Endpoints

| Method | Endpoint | Description | Request | Response |
|---|---|---|---|---|
| `POST` | `/api/v1/ai/suggest` | Improve a draft or existing task | `{ title, description }` | `200` `{ suggestedTitle, suggestedDescription, suggestedPriority, suggestedComplexity, reasoning }` |
| `POST` | `/api/v1/tasks/{id}/ai/breakdown` | Draft subtasks for a task | – | `200` `[{ title, description, priority, complexity }]` |
| `POST` | `/api/v1/ai/chat` | Ask about your tasks | `{ message, history?: [{ role: "user"\|"assistant", content }] }` (at most 10 history items; `message` at most 1000 characters) | `200` `{ reply }` |

## 6. UI scope

All of this follows the conventions in `AGENTS.md`: React Router, TanStack Query with an `src/api/` layer, Tailwind, and react-i18next with EN and PT-BR.

**Routes:**
- `/login`, `/signup`
- `/forgot-password` and `/reset-password?token=` *(Phase 5)*
- `/` (dashboard: task list with filters, sorting, and pagination)
- `/tasks/new`, `/tasks/:id` (detail and edit, with an ancestor breadcrumb and a subtask section)
- the chat side panel, available on every authenticated page

**Auth:**
- The token is stored in `localStorage`.
- An API client adds `Authorization: Bearer <token>` to every request and handles `401` globally.
- Protected routes redirect to `/login`.

**AI UX:**
- Loading states while the AI works.
- A diff-style accept/reject view for suggestions.
- An editable draft list for breakdowns.
- Every error `code` in §2 has an i18n key in both locales.

## 7. Configuration

- New env vars, added to `.env.example`: `JWT_SECRET`, `GEMINI_API_KEY`, `GEMINI_MODEL`, `APP_TIMEZONE`.
- Phase 5 adds `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_FROM`, and `APP_BASE_URL`.
- `application.properties` gets the datasource, Flyway location (`classpath:db/migration/postgres`), Spring AI Google GenAI settings, CORS for `http://localhost:5173`, and `app.tasks.max-depth=5` (not an env var).

## 8. Testing strategy

- Each BDD scenario, including the negative ones, maps to at least one test. Test names follow the scenario, e.g. `signUp_withExistingEmail_returns409`.
- **API:** `@SpringBootTest` integration tests with **Testcontainers PostgreSQL** and MockMvc.
  - The AI model is mocked (`@MockitoBean ChatModel`) to return canned JSON. This covers malformed output (`422`) and exceptions (`503`).
  - The overdue job is tested with a fixed `Clock`.
- **Authorization:** for every task endpoint there is a test that user B gets `404` for user A's task.
- **UI:** `npm run lint && npm run build`. Component tests for forms and AI accept flows are optional in v1.

## 9. Delivery phases

| Phase | Scope | Done when |
|---|---|---|
| 0 | Fix the infra: `docker-compose.yaml` (`services:`, UI build context, healthcheck), `init.sql`, the UI `nginx.conf`, the V1 comma, `application.properties`, `.env.example` | `docker compose up --build` starts all three services and Flyway applies V1 |
| 1 | Auth: sign up, sign in, `/users/me`, JWT security config, error handler, UI login and signup | Tests pass and the UI can log in and out |
| 2 | Task CRUD, filters, subtasks, lookups, the overdue job, the task schema migration (§1), and the UI dashboard and detail pages | All §4 scenarios are covered by tests |
| 3 | AI suggest and breakdown | All §5 scenarios except chat are covered, with the AI mocked |
| 4 | Chat assistant (query-only) | The chat scenarios are covered |
| 5 | Password reset (SMTP, Mailpit in compose) | The reset scenarios are covered |
| Later | Chat tool calling (create and update tasks), refresh tokens, admin features, parent auto-completion, moving a task to another parent (needs a cycle check) | – |
