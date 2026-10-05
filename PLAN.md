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
| Subtasks | **One level only.** A subtask has exactly one parent and can't have its own subtasks. Deleting a parent deletes its subtasks. Subtasks are hidden from the main list by default. They can be created manually or from AI drafts. |
| AI suggestions | "Enhance" and "Estimate" are **merged** into one `POST /api/v1/ai/suggest` endpoint. It takes a draft `{title, description}`, so it works on unsaved tasks too. Breakdown stays task-based. |
| Chat | **Query-only and stateless** in v1. The client may send the last few messages as context, and nothing is persisted. Creating and editing tasks through chat (tool calling) comes in a later phase. |
| Roles | `USER` and `ADMIN` are seeded. Every new account gets `USER`. There are no admin endpoints in v1. Every user sees only their own tasks. |
| User name | Sign-up takes `name`, a display name that isn't unique and maps to `USERS.NAME`. Users sign in with `email`. |

## 1. Schema changes (`V2__task-due-date-and-subtask-rules.sql`)

- `TASK.DUE_DATE DATE NULL`.
- A not-blank check on title: `CHECK (TITLE ~ '\S')`.
- `TASK.USER_ID` becomes `NOT NULL`, because every task belongs to a user.
- `TASK.STATUS_ID` and `TASK.PRIORITY_ID` become `NOT NULL`. The service applies defaults (see §2).
- `TASK_SUBTASK`: add `UNIQUE (SUBTASK_ID)` so each subtask has one parent, and `CHECK (PARENT_TASK_ID <> SUBTASK_ID)`.
- An index on `TASK (USER_ID, STATUS_ID, DUE_DATE)` for listing and filtering.

The service layer enforces the one-level rule: a task that is already a subtask can't become a parent.

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
  "subtasks": [ "SubtaskSummary" ],
  "createdAt": "ISO-8601",
  "updatedAt": "ISO-8601"
}
```

`subtasks` appears only in the single-task response (`GET /tasks/{id}`). List responses leave it out.

### SubtaskSummary object

```json
{
  "id": "uuid",
  "title": "string (1-100)",
  "status": "TODO | IN_PROGRESS | OVERDUE | DONE",
  "priority": "LOW | MEDIUM | HIGH",
  "dueDate": "2026-10-31 | null"
}
```

`SubtaskSummary` is a deliberately trimmed view, holding only what the subtask list on the parent's detail page displays. `description`, `complexity`, `parentTaskId`, and the timestamps are left out to keep `GET /tasks/{id}` small. To get a subtask's full details, call `GET /tasks/{subtaskId}`.

### Validation and defaults

- `title`: required, not blank, at most 100 characters. `description`: required, not blank, at most 500 characters.
- `priority` defaults to `MEDIUM`. `status` defaults to `TODO`. `complexity` defaults to `null`.
- Enum values are the lookup-table **names**. The service resolves names to IDs, and IDs never appear in the API. An unknown name returns `400`.

### Status rules

- A user may set `TODO`, `IN_PROGRESS`, or `DONE`, and may move between them freely (e.g. reopen a `DONE` task).
- Only the system sets `OVERDUE`. If a user sends `OVERDUE`, the API returns `400 INVALID_STATUS`.
- A scheduled job (`@Scheduled`, daily at 00:05 in `app.timezone`) moves tasks with `status IN (TODO, IN_PROGRESS)` and `dueDate < today` to `OVERDUE`.
- If the user changes the `dueDate` of an `OVERDUE` task to today or later, or clears it, the status goes back to `TODO`. Marking an `OVERDUE` task `DONE` is allowed.
- A parent's status is **not** changed automatically by its subtasks in v1.

### Errors

Every error uses RFC 9457 `ProblemDetail`, plus a `code` property in English and an `errors` list for validation failures. The UI maps `code` to an i18n key.

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Bean Validation failed or a filter value is malformed |
| 400 | `INVALID_STATUS` / `INVALID_PRIORITY` / `INVALID_COMPLEXITY` | Unknown enum name, or `OVERDUE` set by a user |
| 400 | `SUBTASK_DEPTH_EXCEEDED` | Adding subtasks to a task that is itself a subtask |
| 401 | `UNAUTHORIZED` | Missing, invalid, or expired token |
| 401 | `BAD_CREDENTIALS` | Wrong email or password at sign-in |
| 404 | `TASK_NOT_FOUND` | The task doesn't exist **or belongs to another user**. The API never reveals that it exists. |
| 409 | `EMAIL_ALREADY_USED` | Sign-up with an existing email |
| 422 | `AI_INVALID_RESPONSE` | The model's output can't be parsed or validated |
| 429 | `AI_RATE_LIMITED` | The per-user AI quota is exceeded |
| 503 | `AI_UNAVAILABLE` | Gemini timed out, rate-limited us, or is offline |

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
  * **Then** they see its full details, including its subtasks.
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
  * **Then** the task **and all of its subtasks** are deleted in one transaction.

* **Add a subtask manually**
  * **Given** a top-level task,
  * **When** the user adds a subtask,
  * **Then** the subtask is created and linked to the parent.

### Endpoints

| Method | Endpoint | Description | Request | Response |
|---|---|---|---|---|
| `POST` | `/api/v1/tasks` | Create a task | `{ title, description, dueDate?, priority?, complexity? }` | `201` Task, with a `Location` header |
| `GET` | `/api/v1/tasks/{id}` | Get one task | – | `200` Task (with `subtasks`) |
| `PUT` | `/api/v1/tasks/{id}` | Replace editable fields | `{ title, description, dueDate, priority, status, complexity }` | `200` Task |
| `PATCH` | `/api/v1/tasks/{id}` | Partial update | Any subset of the `PUT` fields | `200` Task |
| `DELETE` | `/api/v1/tasks/{id}` | Delete the task and its subtasks | – | `204` |
| `GET` | `/api/v1/tasks` | List and filter | See the query parameters below | `200` Page of Task |
| `POST` | `/api/v1/tasks/{id}/subtasks` | Create subtasks (manual or accepted AI drafts) | `[{ title, description, priority?, complexity?, dueDate? }]` (1–10 items) | `201` `[Task]` |
| `GET` | `/api/v1/lookups` | Enum values for the UI | – | `200` `{ priorities, statuses, complexities }` |

**`GET /api/v1/tasks` query parameters:**

- `status`: repeatable, e.g. `status=TODO&status=OVERDUE`
- `priority`, `complexity`: repeatable
- `dueFrom`, `dueTo`: ISO dates, inclusive
- `q`: case-insensitive match on title and description
- `includeSubtasks`: default `false`
- `page`: default `0`
- `size`: default `20`, max `100`
- `sort`: default `dueDate,asc`. Allowed fields: `dueDate`, `priority`, `createdAt`, `title`.

**Page response:** `{ content: [Task], page, size, totalElements, totalPages }`.

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
  * **Given** a saved top-level task,
  * **When** the user asks the AI to break it down,
  * **Then** the AI returns 2–8 draft subtasks, each with a title, description, priority, and complexity.
  * **And** the user can edit, remove, or reorder the drafts, then accept. Accepting calls `POST /tasks/{id}/subtasks` with the remaining drafts.
  * *Negative:* if the task is itself a subtask, the API returns `400 SUBTASK_DEPTH_EXCEEDED`.

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
- `/tasks/new`, `/tasks/:id` (detail and edit, with a subtask section)
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
- `application.properties` gets the datasource, Flyway location (`classpath:db/migration/postgres`), Spring AI Google GenAI settings, and CORS for `http://localhost:5173`.

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
| 2 | Task CRUD, filters, subtasks, lookups, the overdue job, the V2 migration, and the UI dashboard and detail pages | All §4 scenarios are covered by tests |
| 3 | AI suggest and breakdown | All §5 scenarios except chat are covered, with the AI mocked |
| 4 | Chat assistant (query-only) | The chat scenarios are covered |
| 5 | Password reset (SMTP, Mailpit in compose) | The reset scenarios are covered |
| Later | Chat tool calling (create and update tasks), refresh tokens, admin features, parent auto-completion | – |
