# AGENTS.md

Guidance for AI coding agents working in this repository.

**Read and follow [`RULES.md`](RULES.md) first.** It holds the hard rules (e.g. never push, never edit an applied migration).

## Project overview

Planned AI Task Manager is a task manager that uses AI (Google Gemini via Spring AI) to:

1. **Break tasks into subtasks.** Results are stored as tasks in a tree through `TASK.PARENT_TASK_ID`, at most 5 levels deep (PLAN §0).
2. **Estimate priority and complexity.** The AI suggests `PRIORITIES` (LOW/MEDIUM/HIGH) and `COMPLEXITIES` (EASY/MEDIUM/HARD).
3. **Chat assistant.** Users create and query tasks in natural language.

## Repository layout

This root repo is a thin wrapper. The real code lives in two **git submodules**:

| Path | What | Stack |
|---|---|---|
| `AI-Task-Manager-API/` | REST API | Java 25, Spring Boot 4.1, Spring AI 2.0 (Google GenAI), JPA, Flyway, Spring Security, springdoc-openapi, Lombok, PostgreSQL |
| `AI-Task-Manager-UI/` | Web UI | React 19, TypeScript 6, Vite 8, ESLint |
| `docker-compose.yaml` | Local stack: API (8080), UI (5173 → nginx 80), Postgres 18 (5432) | |
| `init-task-manager-db/` | SQL run by the Postgres container on first start | |

## Commands

API (run inside `AI-Task-Manager-API/`):

```bash
./mvnw spring-boot:run      # run locally
./mvnw test                 # tests
./mvnw package -DskipTests  # build jar
```

UI (run inside `AI-Task-Manager-UI/`):

```bash
npm ci
npm run dev       # Vite dev server on :5173
npm run lint
npm run build     # tsc -b && vite build
npm test          # vitest run
```

Full stack (run from the root): `docker compose up --build`

## Definition of done

A change is not done until:

- New features and bug fixes come with tests.
- `./mvnw test` passes for API changes.
- `npm run lint && npm run build && npm test` passes for UI changes.

If a check can't be run, say so explicitly. Never claim it passed.

## API conventions

- Base package: `br.com.planned.api`. Use a **layered-by-type** layout:
  - `controller/`: REST controllers (thin, no business logic)
  - `service/`: business logic, transactions, and AI calls
  - `repository/`: Spring Data JPA repositories
  - `entity/`: JPA entities
  - `dto/`: request/response records (never expose entities from controllers)
  - `config/`: security, AI, OpenAPI, CORS config
  - `exception/`: custom exceptions and a `@RestControllerAdvice` handler
- Validate request DTOs with Jakarta Validation (`@Valid`).
- Keep Gemini/Spring AI access behind a service, so other code doesn't depend on the model provider directly.
- **Auth:** stateless JWT. Login returns a token, and the UI sends `Authorization: Bearer <token>`. Hash passwords with BCrypt. Roles come from the `ROLE` table (`USER`, `ADMIN`).
- The API docs come from springdoc (Swagger UI at `/swagger-ui.html`). Keep endpoints annotated so the docs stay useful.

### Database and migrations

- Flyway migrations live in `src/main/resources/db/migration/postgres/`.
- Planning docs (`PLAN.md`, `TASKS.md`, `plans/`) say that a migration is needed and what it does, never its version number. The agent writing the migration takes the next free `V{n}` at that time, so parallel or reordered work never fights over a number.
- Lookup tables (`PRIORITIES`, `TASK_STATUS`, `COMPLEXITIES`, `ROLE`) are seeded in migrations. Reference their names, not hard-coded IDs.
- Tasks belong to `USER_ID`. Always scope task queries to the authenticated user.
- Timestamp columns are `TIMESTAMP` (no time zone) holding UTC. Entities map them as `Instant`, set from the injected `Clock`, never `LocalDateTime` or `Instant.now()`. `application.properties` sets Hibernate's `preferred_instant_jdbc_type=TIMESTAMP` and `jdbc.time_zone=UTC` so `ddl-auto=validate` accepts them.

### Tests

- Use **Testcontainers** with PostgreSQL for anything that touches the DB. Don't depend on the docker-compose database.
- Mock the AI model in tests. Tests must never call the real Gemini API.

## UI conventions

- React + TypeScript. Use function components and hooks only.
- **Routing:** React Router.
- **Server state:** TanStack Query, with all API calls in a dedicated API layer (e.g. `src/api/`), not inside components.
- **Styling:** Tailwind CSS.
- **i18n:** react-i18next with **English and Portuguese (PT-BR)**. Never hard-code user-facing strings. Add keys to both locales.

## Language

- Write code, identifiers, comments, commit messages, and API error codes in **English**.
- User-facing UI text goes through i18n (EN + PT-BR).

## Configuration and secrets

- Secrets and env vars live in a git-ignored **`.env` at the repo root**, read by docker-compose. Keep a committed `.env.example` up to date with every variable (no real values).
- Variables: `POSTGRES_USER`, `POSTGRES_PASSWORD`, `TASK_MANAGER_DB_URL`, `AI_PROVIDER`, `AI_MODEL`, `AI_API_KEY`, `AI_BASE_URL`, `JWT_SECRET`, `APP_TIMEZONE`, and the optional UI build-time `VITE_API_URL`. `.env.example` describes each one.
- Never commit secrets or print them in logs.

## Git workflow

- Agents may commit (see `RULES.md` for pushing).
- Use **Conventional Commits**: `feat(api): ...`, `fix(ui): ...`, `chore: ...`, `test(api): ...`, `docs: ...`.
- Changes inside a submodule must be committed **in that submodule's repo**. Then commit the updated submodule pointer in the root repo.
- Don't commit build output (`target/`, `dist/`, `node_modules/`).
