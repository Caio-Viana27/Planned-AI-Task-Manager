# Wave 0: Foundations

**Status:** draft · **PLAN:** §0, §7, §8, §9 phase 0

## Goal

`docker compose up --build` starts the database, API, and UI. Flyway applies V1. Each submodule has a working test setup, and the shared pieces that every later wave builds on are in place: error handling, config, the API client, i18n, and routing.

## Out of scope

- Any feature: auth, tasks, AI, chat. Routes and pages are placeholders only.
- Real security rules. Wave 0 only opens what the healthcheck and Swagger need. Wave 1 adds JWT.
- Mail and Mailpit (Wave 5).

## Decisions for this wave

Confirmed by the maintainer on 2026-10-05.

| # | Decision | Outcome |
|---|---|---|
| D1 | Env var names | Keep the names already in use (`TASK_MANAGER_DB_URL`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, ...). Don't switch to `SPRING_DATASOURCE_*`. |
| D2 | `task_manager_db_test` database | Drop it, along with `TASK_MANAGER_DB_TEST_URL`. Tests use Testcontainers (`AGENTS.md`). |
| D3 | Wave 0 security | A minimal `SecurityConfig` that permits `/actuator/health`, `/swagger-ui/**`, and `/v3/api-docs/**`. Without it, the default config returns 401 and the API healthcheck fails. Wave 1 replaces it. |

## Current state

Some of this work was already started (uncommitted as of 2026-10-05):

- Root: `services:` is fixed. The API gets `GEMINI_*` and `JWT_SECRET`. `init.sql` drops `IF NOT EXISTS`. `.env` is git-ignored.
- API: the V1 comma is fixed. `application.properties` has a first pass at config.

Still broken:

- `.env.example` doesn't exist.
- `application.properties` uses `flyway.*` keys. Spring Boot reads `spring.flyway.*`, so the location setting is ignored. The `gemini.*` keys aren't Spring AI properties either.
- Spring Security's default config blocks `/actuator/health` (see D3).
- `AI-Task-Manager-UI/nginx.conf` is missing, so the UI image doesn't build.
- There's no Testcontainers setup, so `./mvnw test` needs a live database.

## Tasks

Run them in order, one at a time. Review and merge each before starting the next.

### 0.1 Root: fix compose and add `.env.example`

- **Repo:** root
- **Changes:**
  - DB healthcheck: `pg_isready -U $${POSTGRES_USER} -d task_manager_db`. The `$$` makes the container's own env var expand at runtime. Drop `TASK_MANAGER_DB`.
  - API healthcheck: hard-code `http://localhost:8080/actuator/health`. It's container-internal, so it doesn't need a variable. Drop `TASK_MANAGER_API_ACTUATOR_URL`.
  - Mount the Postgres volume at `/var/lib/postgresql`, the PG18 image layout.
  - Pass `APP_TIMEZONE` to the API.
  - Apply D2: remove the test database from `init.sql` and `TASK_MANAGER_DB_TEST_URL` from compose.
  - Add `.env.example` with every variable compose reads, using placeholder values and a comment on each one.
- **Acceptance:** `docker compose config -q` succeeds with a `.env` copied from `.env.example`.
- **Review:** the compose diff, and `.env.example` against the compose file. Every `${VAR}` should be listed.
- **Commit:** `fix: correct docker-compose and add .env.example`

### 0.2 API: make the app boot against the compose database

- **Repo:** API
- **Changes:**
  - Commit the V1 comma fix. It's the approved exception in PLAN §0, since V1 has never been applied anywhere.
  - `application.properties`:
    - datasource from `TASK_MANAGER_DB_URL`, `POSTGRES_USER`, `POSTGRES_PASSWORD`
    - `spring.flyway.locations=classpath:db/migration/postgres`. Remove the `flyway.*` keys; Flyway uses the datasource.
    - `spring.jpa.hibernate.ddl-auto=validate`, `spring.jpa.open-in-view=false`
    - Spring AI Google GenAI API key and model, using the property names from the Spring AI 2.0 docs. Remove the `gemini.*` keys.
    - `app.timezone=${APP_TIMEZONE:America/Sao_Paulo}`, `app.jwt.secret=${JWT_SECRET}`, `app.jwt.ttl=60m`, `app.ai.timeout=20s`, `app.ai.quota-per-hour=30`, `app.cors.allowed-origins=http://localhost:5173`
    - `management.endpoints.web.exposure.include=health`
  - `config/SecurityConfig` for D3.
  - Confirm the runtime image (`ubi10-minimal`) has `curl` for the healthcheck. If not, install it in the Dockerfile.
- **Acceptance:** `docker compose up --build task-manager-db task-manager-api` reports both healthy. The API logs show Flyway applying V1.
- **Review:** the properties file, line by line. This is where every later wave reads config from.
- **Commit:** `chore(api): configure datasource, flyway, ai and app properties`

### 0.3 UI: make the image build

- **Repo:** UI
- **Changes:**
  - Add `nginx.conf` with an SPA fallback (`try_files $uri /index.html`) and a proxy from `/api/` to `http://task-manager-api:8080`.
  - Remove the Vite demo content (`App.css` and the counter).
- **Acceptance:** `docker build .` succeeds.
- **Commit:** `fix(ui): add nginx config and remove vite demo`

> **Checkpoint A:** `docker compose up --build` from the root starts all three services healthy. `http://localhost:5173` serves the UI and `http://localhost:8080/swagger-ui.html` loads.

### 0.4 API: Testcontainers harness

- **Repo:** API
- **Changes:**
  - `pom.xml`: `spring-boot-testcontainers`, `org.testcontainers:postgresql`, and `org.testcontainers:junit-jupiter` (test scope).
  - `src/test/java/br/com/planned/api/support/IntegrationTest.java`: `@SpringBootTest` with a shared `@ServiceConnection` PostgreSQL 18 container, and `@MockitoBean ChatModel`.
  - `src/test/resources/application-test.properties`: a test JWT secret and a fake Gemini key.
  - Make `contextLoads` extend the base class.
- **Acceptance:** `./mvnw test` passes with only Docker running, with no compose database and no `.env`. No test reaches Gemini.
- **Commit:** `test(api): add testcontainers integration test harness`

### 0.5 API: error handling

- **Repo:** API · **PLAN:** §2 (Errors)
- **Changes:**
  - `exception/ErrorCode`: an enum with every code in PLAN §2, each with its HTTP status.
  - `exception/ApiException`, which carries an `ErrorCode`.
  - `exception/GlobalExceptionHandler` (`@RestControllerAdvice`). It returns a `ProblemDetail` with a `code` property and, for validation failures, an `errors: [{field, message}]` list. It handles `ApiException`, `MethodArgumentNotValidException`, `HandlerMethodValidationException`, `MethodArgumentTypeMismatchException`, and unreadable bodies. Spring MVC's own errors (405, 415, missing parameters) get a `code` from their status, and any other exception becomes a logged, generic 500 `INTERNAL_ERROR`.
- **Acceptance:** a test controller in `src/test` triggers each exception type, including 405, 415, and 500, and the tests check the status, `code`, and `errors`.
- **Commit:** `feat(api): add problem-detail error handling`

### 0.6 API: app config beans

- **Repo:** API · **PLAN:** §0 ("Today"), §7
- **Changes:**
  - `config/AppProperties`: a `@ConfigurationProperties` record for `app.*`.
  - A `Clock` bean in `app.timezone`.
  - `config/OpenApiConfig` with a bearer JWT security scheme.
  - `config/CorsConfig` from `app.cors.allowed-origins`.
- **Acceptance:** tests show `AppProperties` binds, the `Clock` uses the configured zone, and a preflight request from `http://localhost:5173` gets CORS headers.
- **Commit:** `feat(api): add app properties, clock, openapi and cors config`

### 0.7 UI: add the stack

- **Repo:** UI
- **Changes:**
  - Install `react-router`, `@tanstack/react-query`, `tailwindcss` and `@tailwindcss/vite`, and `i18next`, `react-i18next`, `i18next-browser-languagedetector`.
  - Dev: `vitest`, `@testing-library/react`, `@testing-library/user-event`, `jsdom`. Add the script `"test": "vitest run"`.
  - Set up Tailwind, and add the Vite dev proxy `/api` → `http://localhost:8080`.
  - `src/main.tsx` wires up `QueryClientProvider`.
  - One smoke test that renders `App`.
- **Acceptance:** `npm run lint && npm run build && npm test` passes, and a Tailwind class visibly applies in `npm run dev`.
- **Review:** `package.json` only. This is the one place in Wave 0 that adds UI dependencies.
- **Commit:** `chore(ui): add router, query, tailwind, i18n and vitest`

### 0.8 UI: API client

- **Repo:** UI · **PLAN:** §2 (Errors)
- **Changes:**
  - `src/auth/tokenStorage.ts`: `localStorage` access wrapped in `try/catch`.
  - `src/api/client.ts`: a fetch wrapper.
    - base URL from `VITE_API_URL`, defaulting to `/api`
    - adds the `Authorization` header when a token exists
    - parses a `ProblemDetail` into `ApiError { status, code, errors }`
    - calls a registered `onUnauthorized` handler on `401`
- **Acceptance:** unit tests show the client adds the header, parses a ProblemDetail, calls the 401 hook, and survives `localStorage` throwing.
- **Commit:** `feat(ui): add api client and token storage`

### 0.9 UI: i18n, router, and layout

- **Repo:** UI · **PLAN:** §6
- **Changes:**
  - `src/i18n/index.ts`, plus `src/i18n/locales/{en,pt-BR}/{common,errors}.json`. `errors.json` has a key for every code in PLAN §2.
  - `src/routes/router.tsx` with every route from PLAN §6, each pointing to a placeholder in `src/pages/*Page.tsx`.
  - A `ProtectedRoute` stub that renders its children (Wave 1 makes it real).
  - `AppLayout` with a header, an EN / PT-BR language switcher, and an empty slot for the chat panel.
- **Acceptance:** tests show each route renders its placeholder and that switching language changes the visible text. Both locale files have the same keys.
- **Commit:** `feat(ui): add i18n, router and app layout`

> **Checkpoint B (wave done):** run `docker compose up --build` again. Click through every placeholder route in both languages. Run `./mvnw test` in the API and `npm run lint && npm run build && npm test` in the UI. Then commit the submodule pointers in the root: `chore: bump submodules (wave 0)`.

## After this wave

- Remove the "Known issues" section from `AGENTS.md` once 0.2 and 0.3 fix the rest. The env var list and the `UI` folder name were updated after 0.1.
- Add `npm test` to the UI commands in `AGENTS.md`.
