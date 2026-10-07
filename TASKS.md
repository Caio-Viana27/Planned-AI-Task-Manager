# TASKS.md

This file breaks [`PLAN.md`](PLAN.md) into tasks that an orchestrator can hand out to a team of agents. Each task is written so an agent can start cold. It should read `RULES.md`, `AGENTS.md`, the `PLAN.md` sections the task cites, and the task card itself.

- `PLAN.md` is the source of truth for behavior and API contracts.
- This file covers ordering, ownership, and acceptance.
- If the two disagree, `PLAN.md` wins. Report the conflict to the orchestrator.

## How to orchestrate

### Roles

- **Orchestrator** (one agent, the main session):
  - Hands out tasks and reviews results.
  - Merges each finished task branch into `main` in the submodule.
  - Commits submodule pointers in the root repo.
  - Is the only agent that edits root-repo files (`docker-compose.yaml`, `.env.example`, `AGENTS.md`, `PLAN.md`, `TASKS.md`).
- **Worker** (one agent per task):
  - Works in **one** submodule only.
  - Uses its own git worktree of that submodule on branch `task/<ID>` (e.g. `task/T2.3`), created from the submodule's current `main`.
  - Commits there, following the Conventional Commits rule in `AGENTS.md`.
  - Never pushes (`RULES.md`).

### Steps for each task

1. **Check that it can start.** Every task in `Depends on` must already be merged into the submodule's `main`.
2. **Hand it out.** Give the worker:
   - the task card
   - the paths to `RULES.md`, `AGENTS.md`, and `PLAN.md`
   - the worktree path
3. **The worker works.** It implements the task, writes tests, and runs the task's *Verify* commands. It reports:
   - its commits
   - the output of the test, lint, and build commands
   - any deviations from `PLAN.md`
4. **Review and merge.** The orchestrator reviews the result, then merges with `--no-ff` into the submodule's `main`, running *Verify* again after the merge.
5. **Commit the pointer.** At the end of each wave, the orchestrator commits the updated submodule pointers in the root repo: `chore: bump submodules (wave N)`.

### Shared hotspot files

Several parallel tasks have to touch the same files. The rules for each:

| File | Rule |
|---|---|
| `pom.xml`, `package.json` | Only Wave 0 tasks, T1.2 (the OAuth2 resource server starter), and T5.2 add dependencies. Any other task that needs one stops and asks the orchestrator. |
| `exception/ErrorCode.java` | Every code in PLAN §2 is created up front in T0.4. Later tasks only *use* the codes. |
| `application.properties` | T0.2 adds every property the plan needs, including AI, JWT, and timezone. T1.1 adds the two Hibernate timestamp properties (wave 1 plan, D3). T2.1 adds `app.tasks.max-depth=5` and its `AppProperties` binding. Later tasks only read them. |
| UI locale files `src/i18n/locales/{en,pt-BR}/*.json` | One **namespace file per feature** (`auth.json`, `tasks.json`, `ai.json`, `aiBreakdown.json`, `chat.json`, `passwordReset.json`, `errors.json`), so parallel tasks don't touch the same file. `errors.json` (all error codes) is created up front in T0.3. T1.7 adds `auth.json` and registers it in `src/i18n/index.ts`, `i18next.d.ts`, and `locales.test.ts`; later namespaces are registered the same way. |
| UI router `src/routes/router.tsx` | T0.3 creates every route as a placeholder page. T1.5 wraps the routes in a root auth route. Other feature tasks replace their own page files and never edit the router. |
| `config/SecurityConfig.java` | T1.3 owns this file. T5.2 is the only later task that may touch it, to make the reset endpoints public. |

### Definition of done for every task

- The task's acceptance criteria are met, with tests (`AGENTS.md`).
- API tasks: `./mvnw test` passes.
- UI tasks: `npm run lint && npm run build && npm test` passes.
- The worker reports any check it could not run and never claims it passed.
- Nothing is pushed.

### Before Wave 0

Settle the root repo state with the maintainer. The root currently has staged `.gitmodules` and submodules, plus untracked `docker-compose.yaml`, `init-task-manager-db/`, and `.vscode/`. T0.1 assumes the maintainer has decided whether these get committed, and that `.vscode/` stays out.

---

## Dependency waves

Tasks in the same wave can run in parallel once their dependencies are merged.

```
Wave 0  T0.1 (root) · T0.2 (API) · T0.3 (UI)
        T0.4 (API, after T0.2)
Wave 1  T1.1 → T1.2 → T1.3 → T1.4 (API)  ‖  T1.5 → T1.6 → T1.7 (UI)   (see plans/wave-1-auth.md)
Wave 2  T2.1 (API) → T2.2 (API) → { T2.3, T2.4, T2.5 } (API)
        T2.6 (UI) → { T2.7, T2.8 } (UI)   (see plans/wave-2-tasks.md)
        Addendum D9: T2.9 (API, after T2.2) → T2.10 (UI, after T2.7, T2.8)
Wave 3  T3.1 (API) → { T3.2, T3.3 } (API)
        { T3.4, T3.5 } (UI, after T2.8)
Wave 4  T4.1 (API) ‖ T4.2 (UI)
Wave 5  T5.1 (root) → T5.2 (API) ‖ T5.3 (UI)
Final   T9.1 (root, orchestrator)
```

UI tasks can run alongside API tasks in the same phase, because they code against the contract in PLAN.md, not against the running API. To develop a UI task without the API, the worker may use mocked responses in tests. Don't add MSW or any other mock-server dependency without asking.

---

## Wave 0: Foundations

### T0.1 Fix root infra
- **Repo:** root (orchestrator, or a worker with root access) · **Depends on:** none · **PLAN:** §7, §9 phase 0
- **Scope:**
  - `docker-compose.yaml`:
    - Rename `service:` to `services:`.
    - Rename the API service `user-service-api` to `task-manager-api`.
    - Point the UI service build at `./AI-Task-Manager-UI`.
    - Pass `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`, `JWT_SECRET`, `GEMINI_API_KEY`, `GEMINI_MODEL`, and `APP_TIMEZONE` to the API.
    - Healthcheck: `pg_isready -U $${POSTGRES_USER} -d task_manager_db`.
    - Postgres 18 volume: mount `/var/lib/postgresql`, which is the PG18 image layout.
    - The UI depends on the API.
  - `init-task-manager-db/init.sql`: Postgres has no `CREATE DATABASE IF NOT EXISTS`. The init scripts run only on an empty volume, so plain `CREATE DATABASE task_manager_db;` is fine. Drop `task_manager_db_test`, since tests use Testcontainers.
  - Add a root `.gitignore` containing `.env` and `.vscode/`.
  - Add `.env.example` listing every variable from PLAN §7, without the Phase 5 mail variables, using placeholder values.
- **Acceptance:** `docker compose config` succeeds. Once T0.2 and T0.3 are merged, `docker compose up --build` starts the database, the API, and the UI. The API container logs show Flyway applying V1.
- **Verify:** `docker compose config -q`. The full `up --build` check runs again in T9.1.
- **Commit:** `fix: correct docker-compose, db init script and add .env.example`

### T0.2 API: build, config, and test harness
- **Repo:** API · **Depends on:** none · **PLAN:** §0 (V1 migration), §7, §8
- **Scope:**
  - Fix the missing comma in `V1__create-tables.sql`. This is the approved exception noted in PLAN §0.
  - `pom.xml`: add these dependencies:
    - `spring-boot-starter-oauth2-resource-server`
    - `org.testcontainers:postgresql` and `junit-jupiter`, plus `spring-boot-testcontainers` (test scope)
  - `application.properties`:
    - datasource from `SPRING_DATASOURCE_*` env vars
    - `spring.jpa.hibernate.ddl-auto=validate`
    - `spring.jpa.open-in-view=false`
    - Flyway location `classpath:db/migration/postgres`
    - Spring AI Google GenAI: API key `${GEMINI_API_KEY}`, model `${GEMINI_MODEL:gemini-2.5-flash}`
    - `app.timezone=${APP_TIMEZONE:America/Sao_Paulo}`
    - `app.jwt.secret=${JWT_SECRET}`, `app.jwt.ttl=60m`
    - `app.ai.timeout=20s`, `app.ai.quota-per-hour=30`
    - `app.cors.allowed-origins=http://localhost:5173`
  - A test base class `src/test/java/.../support/IntegrationTest.java`:
    - `@SpringBootTest` with a `@ServiceConnection` PostgreSQL 18 Testcontainer, shared by all tests
    - `@MockitoBean ChatModel`
    - a test JWT secret and a fake Gemini key in `src/test/resources/application-test.properties`
  - Make `contextLoads` extend the base class.
- **Acceptance:** `./mvnw test` passes with Docker available. Flyway applies V1 inside the container. No test reaches Gemini.
- **Verify:** `./mvnw test`
- **Commit:** `chore(api): configure datasource, flyway, ai and testcontainers harness`

### T0.3 UI: scaffold the stack
- **Repo:** UI · **Depends on:** none · **PLAN:** §6
- **Scope:**
  - Add `nginx.conf`: SPA fallback (`try_files $uri /index.html`) and a proxy from `/api/` to `http://task-manager-api:8080`.
  - Install:
    - `react-router`
    - `@tanstack/react-query`
    - `tailwindcss` with `@tailwindcss/vite`
    - `i18next`, `react-i18next`, `i18next-browser-languagedetector`
    - dev: `vitest`, `@testing-library/react`, `@testing-library/user-event`, `jsdom`
  - Add the script `"test": "vitest run"`.
  - Remove the Vite demo content (`App.css`, the counter).
  - Create this structure:
    - `src/api/client.ts`: a fetch wrapper.
      - Base URL from `VITE_API_URL`, defaulting to `/api`.
      - Adds the `Authorization` header from `src/auth/tokenStorage.ts`.
      - Parses `ProblemDetail` into an `ApiError { status, code, errors }`.
      - Calls a registered `onUnauthorized` handler on `401`.
    - `src/auth/tokenStorage.ts`: `localStorage` access wrapped in `try/catch`.
    - `src/i18n/index.ts`, plus `src/i18n/locales/{en,pt-BR}/{common,errors}.json`. `errors.json` has a key for **every** error code in PLAN §2.
    - `src/routes/router.tsx`, with every route from PLAN §6 pointing to placeholder pages in `src/pages/*Page.tsx`, a `ProtectedRoute` stub, and an `AppLayout` with a header and a slot for the chat panel.
    - `src/main.tsx`: wires up `QueryClientProvider`, `RouterProvider`, and i18n.
    - A language switcher (EN / PT-BR) in the header.
  - Vite dev proxy: `/api` → `http://localhost:8080`.
- **Acceptance:**
  - Every route renders its placeholder.
  - Switching language changes the visible text.
  - `client.ts` has unit tests: it adds the header, parses a ProblemDetail, and calls the 401 hook.
  - The Docker image builds.
- **Verify:** `npm ci && npm run lint && npm run build && npm test && docker build .`
- **Commit:** `feat(ui): scaffold router, query, tailwind, i18n and api client`

### T0.4 API: cross-cutting foundation
- **Repo:** API · **Depends on:** T0.2 · **PLAN:** §2 (Errors), §5 (shared rules: config), §7
- **Scope:**
  - `exception/ErrorCode` enum. Each entry has an HTTP status and is one of every code in PLAN §2.
  - `exception/ApiException`, which carries an `ErrorCode`.
  - `exception/GlobalExceptionHandler` (`@RestControllerAdvice`). It returns a `ProblemDetail` with a `code` property and, for validation failures, an `errors: [{field, message}]` list. It handles:
    - `ApiException`
    - `MethodArgumentNotValidException`
    - `HandlerMethodValidationException`
    - `MethodArgumentTypeMismatchException`
    - unreadable request bodies
    - Spring MVC's own errors (405, 415, missing parameters, ...), with a `code` from their status
    - any other exception, as a generic 500 `INTERNAL_ERROR` that is logged but never exposed
  - `config/AppProperties`: a `@ConfigurationProperties` record for `app.*`.
  - A `Clock` bean in `app.timezone`, injected everywhere a "now" or "today" is needed.
  - `config/OpenApiConfig`: a bearer JWT security scheme.
  - `config/CorsConfig`.
- **Acceptance:**
  - Slice tests show that each exception type produces the documented status and `code`.
  - Swagger UI loads at `/swagger-ui.html`. Security is still the default until T1.2, so permit it in a temporary test config if needed.
- **Verify:** `./mvnw test`
- **Commit:** `feat(api): add problem-detail error handling, app properties, clock and openapi config`

## Wave 1: Auth (PLAN §3)

The task cards for this wave live in [`plans/wave-1-auth.md`](plans/wave-1-auth.md), together with its decisions (D1–D7). That plan replaces the cards that used to be here. `T1.n` anywhere in this file means task 1.n of that plan:

| ID | Task | Repo |
|---|---|---|
| T1.1 | User and role persistence, plus the case-insensitive email migration | API |
| T1.2 | JWT issuing and decoding | API |
| T1.3 | JWT security (owns `SecurityConfig`) | API |
| T1.4 | Auth and current-user endpoints | API |
| T1.5 | Auth API and session provider | UI |
| T1.6 | Protected routes and session-aware header | UI |
| T1.7 | Login and signup pages | UI |

## Wave 2: Tasks (PLAN §1, §2, §4)

The task cards for this wave live in [`plans/wave-2-tasks.md`](plans/wave-2-tasks.md), together with its decisions (D1–D8). That plan replaces the cards that used to be here. `T2.n` anywhere in this file means task 2.n of that plan:

| ID | Task | Repo |
|---|---|---|
| T2.1 | Task schema, entities, and lookups | API |
| T2.2 | Task CRUD | API |
| T2.3 | List, filter, sort, and paginate | API |
| T2.4 | Create subtasks | API |
| T2.5 | Overdue job | API |
| T2.6 | Task API layer and shared components | UI |
| T2.7 | Dashboard | UI |
| T2.8 | Task form, detail, and subtasks | UI |
| T2.9 | Completing a task completes its subtree (addendum, D9) | API |
| T2.10 | Refresh subtasks after a task is marked done (addendum, D9) | UI |

## Wave 3: AI suggest and breakdown (PLAN §5)

### T3.1 API: AI foundation
- **Repo:** API · **Depends on:** T0.4, T1.3 · **PLAN:** §5 (Shared rules)
- **Owns:** `service/ai/AiClient`, `service/ai/AiQuotaService`, `config/AiConfig`, and `src/main/resources/prompts/`.
- **Scope:**
  - `AiClient`: the **only** class that uses `ChatClient`. It exposes a generic `<T> T call(String systemPrompt, Map<String,Object> userData, Class<T> type, Locale locale)`, which:
    - adds today's date, the timezone, and the locale to every prompt
    - puts user and task text inside delimited data sections
    - converts the response into structured output, then runs Bean Validation on it; on failure → `AI_INVALID_RESPONSE`
    - applies the timeout (`app.ai.timeout`) and one retry, only on transient errors
    - maps timeouts, 429, and 5xx from the provider to `AI_UNAVAILABLE`
  - `AiQuotaService`: an in-memory sliding window per user (`app.ai.quota-per-hour`). Exceeding it → `AI_RATE_LIMITED`. A single in-memory instance is fine, because the app is a monolith.
- **Acceptance:** tests with the mocked `ChatModel`:
  - valid JSON → a parsed record
  - malformed JSON or an invalid enum → 422
  - an exception or timeout → 503, with one retry
  - the 31st call within an hour → 429 (fixed `Clock`)
  - the prompt contains the date and the locale (capture the `Prompt` argument)
- **Verify:** `./mvnw test`
- **Commit:** `feat(api): add ai client with structured output, resilience and quota`

### T3.2 API: `/ai/suggest`
- **Repo:** API · **Depends on:** T3.1 · **Parallel with:** T3.3 · **PLAN:** §5 (Suggest scenario, endpoint)
- **Owns:** `service/ai/TaskSuggestionService`, `controller/AiController` (only the suggest method for now), `dto/SuggestRequest`, `dto/SuggestResponse`, and `prompts/suggest.st`.
- **Scope:**
  - Validated input: title of at most 100 characters and description of at most 500, both not blank.
  - The response's enums must be valid lookup names, and the suggested title and description must respect the same length limits.
  - Locale comes from the `Accept-Language` header (`en` or `pt-BR`, defaulting to `en`).
  - The endpoint makes no database writes.
- **Acceptance:**
  - Success with a mocked model.
  - Invalid input → 400.
  - A model response with a 300-character title → 422.
  - No `TASK` rows change.
  - Unauthenticated → 401.
- **Verify:** `./mvnw test`
- **Commit:** `feat(api): add ai task suggestion endpoint`

### T3.3 API: breakdown
- **Repo:** API · **Depends on:** T3.1, T2.4 · **Parallel with:** T3.2 · **PLAN:** §5 (Breakdown scenario, endpoint)
- **Owns:** `service/ai/TaskBreakdownService`, `controller/TaskAiController` (`POST /api/v1/tasks/{id}/ai/breakdown`), and `prompts/breakdown.st`.
- **Scope:**
  - Load the caller's task: 404 if not found or not theirs; 400 `SUBTASK_DEPTH_EXCEEDED` if it is at the maximum depth (T2.2's depth helper).
  - Put the ancestors' titles, root first, in the prompt's data section as context.
  - Return 2–8 drafts, each with a valid priority and complexity.
  - The endpoint makes no database writes.
- **Acceptance:**
  - Success.
  - Breaking down a depth-3 task succeeds, and the captured prompt contains its ancestors' titles.
  - The 404, depth (a depth-5 task), 422 (1 draft or 9 drafts), and 503 cases.
  - No rows are written.
- **Verify:** `./mvnw test`
- **Commit:** `feat(api): add ai subtask breakdown endpoint`

### T3.4 UI: AI suggest UX
- **Repo:** UI · **Depends on:** T2.8 · **Parallel with:** T3.5 · **PLAN:** §5 (Suggest scenario), §6 (AI UX)
- **Owns:** `src/api/ai.ts`, `src/features/ai/suggest/**`, `src/i18n/locales/{en,pt-BR}/ai.json`, and the `AiSuggestSlot` implementation.
- **Scope:**
  - A "Suggest with AI" button inside `TaskForm`, for both create and edit, enabled once the title and description are filled in.
  - A side-by-side current-vs-suggested view, showing the reasoning, with per-field checkboxes and the actions "Accept all", "Accept selected", and "Dismiss".
  - Accepting:
    - in create mode, fills the form
    - in edit mode, calls `PATCH` with only the selected fields
  - Loading state, and localized messages for 422, 429, and 503.
  - The form keeps its values when the AI fails.
- **Acceptance:** component tests:
  - selecting fields and then accepting patches only those fields
  - a 503 shows the unavailable message and leaves the form intact
- **Verify:** `npm run lint && npm run build && npm test`
- **Commit:** `feat(ui): add ai suggestion review and accept flow`

### T3.5 UI: AI breakdown UX
- **Repo:** UI · **Depends on:** T2.8 · **Parallel with:** T3.4 · **PLAN:** §5 (Breakdown scenario), §6 (AI UX)
- **Owns:** `src/api/aiBreakdown.ts`, `src/features/ai/breakdown/**`, and `src/i18n/locales/{en,pt-BR}/aiBreakdown.json`. It is kept separate from T3.4's `ai.ts` and `ai.json` so the two tasks don't edit the same files.
- **Scope:**
  - A "Break down with AI" button in the subtask section, hidden when `canAddSubtasks` is `false`.
  - The drafts appear as an editable list: edit the title, description, and priority; remove; reorder.
  - "Create N subtasks" calls `POST /tasks/{id}/subtasks`, invalidates the task query, and closes the panel.
  - The button is disabled while the request is in flight, to prevent duplicate accepts.
  - AI error states, as in T3.4.
- **Acceptance:** component tests:
  - editing and removing a draft sends the edited list
  - a 503 shows the localized message
- **Verify:** `npm run lint && npm run build && npm test`
- **Commit:** `feat(ui): add ai subtask breakdown review and accept flow`

## Wave 4: Chat (PLAN §5, chat)

### T4.1 API: chat
- **Repo:** API · **Depends on:** T3.1, T2.2 · **PLAN:** §0 (Chat), §5 (Chat scenario, endpoint)
- **Owns:** `service/ai/ChatAssistantService`, the chat method in `controller/AiController`, `dto/ChatRequest`, `dto/ChatResponse`, `prompts/chat.st`, and a repository query for the chat context.
- **Scope:**
  - Validation: `message` is at most 1000 characters; `history` has at most 10 items with roles `user` or `assistant`.
  - Context: up to 20 of the caller's tasks that aren't DONE, ordered OVERDUE first, then `dueDate` ascending with nulls last, then priority descending.
  - The system prompt says:
    - the assistant is read-only
    - it answers only from the provided tasks
    - for requests to create or edit tasks, it explains how to do so in the UI
  - The reply is in the request's locale.
- **Acceptance:**
  - The captured prompt contains only the caller's tasks, in the right order, capped at 20 (seed 25).
  - Validation → 400.
  - A model failure → 503.
- **Verify:** `./mvnw test`
- **Commit:** `feat(api): add read-only chat assistant endpoint`

### T4.2 UI: chat panel
- **Repo:** UI · **Depends on:** T1.6 (layout slot) · **PLAN:** §5 (Chat scenario), §6
- **Owns:** `src/api/chat.ts`, `src/features/chat/**`, and `chat.json`.
- **Scope:**
  - A collapsible side panel in the `AppLayout` slot.
  - The message list is kept in component state only, not persisted. Each request sends the last 10 messages as history.
  - While waiting for a reply, show a typing indicator and disable sending.
  - If sending fails, the input text is kept and a localized error appears (503 or 429).
  - Clear the chat on logout.
- **Acceptance:** component tests:
  - the request includes the last 10 messages of history
  - when sending fails, the input text is still there and the error shows
- **Verify:** `npm run lint && npm run build && npm test`
- **Commit:** `feat(ui): add chat assistant panel`

## Wave 5: Password reset (PLAN §3, Phase 5)

### T5.1 Root: mail infra
- **Repo:** root (orchestrator) · **Depends on:** T0.1 · **PLAN:** §7
- **Scope:**
  - Add a `mailpit` service to `docker-compose.yaml` (SMTP 1025, UI 8025).
  - Pass the `MAIL_*` and `APP_BASE_URL` variables to the API.
  - Add them to `.env.example`.
- **Verify:** `docker compose config -q`
- **Commit:** `chore: add mailpit and mail env vars`

### T5.2 API: password reset
- **Repo:** API · **Depends on:** T1.4, T5.1 · **PLAN:** §3 (Reset password scenario and endpoints)
- **Scope:**
  - Add the `spring-boot-starter-mail` dependency.
  - A new migration for the `PASSWORD_RESET_TOKEN` table.
  - `entity/PasswordResetToken` and its repository.
  - `service/PasswordResetService`:
    - creates a random 32-byte token, stores only its SHA-256 hash, sets a 30-minute expiry, and makes it single-use
    - creating a new token invalidates the user's earlier ones
    - emails a link to `${APP_BASE_URL}/reset-password?token=...`
    - rate-limits requests per email and per IP (in memory)
  - Endpoints:
    - `POST /api/v1/auth/password-reset`: always returns 200, with no difference in the body for unknown emails
    - `PUT /api/v1/auth/password-reset`: returns 204
  - Make both endpoints public in `SecurityConfig`.
  - Add the error code `INVALID_RESET_TOKEN` (400) to `ErrorCode`. This is the one approved later addition to the hotspot. The worker reports this code to the orchestrator, who adds it to PLAN §2.
- **Acceptance:** tests using a mocked `JavaMailSender`:
  - a known email sends mail
  - an unknown email sends nothing but gets the same 200
  - a valid token changes the password, and the old password stops working
  - expired, used, and wrong tokens → 400
  - the rate limit applies
- **Verify:** `./mvnw test`
- **Commit:** `feat(api): add password reset flow`

### T5.3 UI: password reset pages
- **Repo:** UI · **Depends on:** T1.7 · **Parallel with:** T5.2 · **PLAN:** §3, §6
- **Owns:** `ForgotPasswordPage`, `ResetPasswordPage`, `src/api/passwordReset.ts`, and a new `src/i18n/locales/{en,pt-BR}/passwordReset.json`.
- **Scope:**
  - The forgot page always shows a neutral "if the email exists, we sent a link" message.
  - The reset page reads `token` from the URL, takes the new password and a confirmation, and on success redirects to `/login` with a success notice.
  - Add a "Forgot password?" link on the login page. This is a one-line edit to `LoginPage`; the orchestrator merges it.
  - Add `errors.INVALID_RESET_TOKEN` to both locales.
- **Acceptance:** component tests for both pages, including the invalid-token error.
- **Verify:** `npm run lint && npm run build && npm test`
- **Commit:** `feat(ui): add forgot and reset password pages`

## Final

### T9.1 Integration check and docs (orchestrator)
- **Repo:** root · **Depends on:** all tasks in the scope being shipped
- **Scope:**
  - Fill in `.env` locally from `.env.example`, then run `docker compose up --build`.
  - Smoke test in the browser:
    - sign up
    - create a task
    - filter
    - AI suggest
    - breakdown and accept
    - chat
    - logout
    - the PT-BR switch
  - AI steps need a real `GEMINI_API_KEY`. If none is available, say so and mark them as skipped, never as passed.
  - Update `AGENTS.md`:
    - env var list
    - `npm test` command
    - the v1 chat scope, which is query-only
    - remove fixed "Known issues"
  - Commit the final submodule pointers.
- **Commit:** `docs: update agents guide after v1 implementation` and `chore: bump submodules`
