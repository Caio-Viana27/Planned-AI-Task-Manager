# Wave 3: AI suggest and breakdown

**Status:** planned, all decisions confirmed (2026-10-07) · **PLAN:** §0 (Subtasks), §2 (Errors: 422, 429, 503), §5 (shared rules, Suggest and Breakdown scenarios and endpoints), §6 (AI UX), §8, §9 phase 3

## Goal

A logged-in user can ask the AI to improve a task. Gemini rewrites the title and description and suggests a priority and complexity, and the user accepts some, all, or none of the fields in the task form. On a saved task below the maximum depth, the user can ask for 2–8 draft subtasks, edit, remove, or reorder them, and create them in one click. The AI never writes to the database. Every AI call is time-limited, retried at most once, and counted against a per-user hourly quota. Every failure becomes `422`, `429`, or `503` with a localized message, and the user's work stays intact. Tests never reach Gemini.

## Out of scope

- The chat assistant (Wave 4). T3.1's `AiClient` is built so Wave 4 can reuse it.
- Streaming responses, and a per-user or per-request choice of model.
- Storing suggestions or drafts. They live only in the UI until accepted.
- A distributed or persistent quota. One in-memory instance is enough for the monolith (PLAN §5).
- A `Retry-After` header on `429 AI_RATE_LIMITED`.
- Tests against the real Gemini API. The manual checkpoint uses a real key only when one is available.

## Decisions for this wave

Confirmed by the maintainer on 2026-10-07. D10 was added and confirmed the same day as an addendum. D1, D2, D5, and D7 change PLAN wording, so the orchestrator folds them into PLAN after the wave (D10 is already in PLAN §5 and §7). Details below the table.

| # | Decision | Outcome |
|---|---|---|
| D1 | One provider-neutral AI client | `service/ai/AiClient` is the only class that uses `ChatClient` (PLAN §5 calls it `AiAssistantService`; this wave settles on `AiClient`). Provider-specific code exists in only three places: the starter in `pom.xml`, the `spring.ai.*` properties, and `AiClient.isTransient`. See below. |
| D2 | Retry and timeout | Spring AI's built-in retry is turned off. `AiClient` alone allows one retry, and a single `app.ai.timeout` deadline (20 s) covers both attempts. A timeout isn't retried. See below. |
| D3 | Quota accounting | One unit per request that reaches the model, counted after validation, ownership, and depth checks. A failed AI call still counts. A retry doesn't count again. See below. |
| D4 | Prompt layout | System prompt from a `prompts/*.st` template, in English. User and task text are sent only as JSON inside `<data>…</data>`, with `<` and `>` escaped so the delimiter can't be forged. Prompts are never logged. See below. |
| D5 | Locale | The API maps `Accept-Language` to `en` or `pt-BR` (any `pt*` → `pt-BR`, anything else or missing → `en`). The UI sends its current i18n language as `Accept-Language` on every request, not the browser's default. |
| D6 | Validating AI output | Response records carry Bean Validation (non-blank, lengths, list sizes). Lookup names are `String`s that the feature service checks against `LookupService`. Any failure → `422 AI_INVALID_RESPONSE`. The prompt states every limit. See below. |
| D7 | Accepting a suggestion fills the form | In both create and edit mode, accepting writes the chosen fields into `TaskForm`, and the user saves as usual (`POST`, or `PATCH` with the changed fields). The AI flow never calls `PATCH` itself. See below. |
| D8 | Breakdown context | The prompt gets the task's title and description, its ancestors' titles (root first), and its current direct subtasks' titles, so drafts don't repeat existing work. Still 2–8 drafts. |
| D9 | AI calls in the UI | AI requests are TanStack mutations: never cached, never auto-retried (each try costs quota), and the trigger button is disabled while one is in flight. |
| D10 | Vendor-neutral env vars (addendum) | `GEMINI_API_KEY` and `GEMINI_MODEL` become `AI_PROVIDER`, `AI_MODEL`, `AI_API_KEY`, and `AI_BASE_URL`. `AI_PROVIDER` selects the Spring AI chat provider. Gemini is the only one built in, and adding another is a short checklist. See below. |

### D1: provider-neutral AI client

- **Structured output** uses `ChatClient`'s `.entity(Class<T>)` (Spring AI's `BeanOutputConverter`), which works with any provider. Don't use Gemini-only options such as a native response schema or `GoogleGenAiChatOptions`.
- **Error classification** lives in one private method, `AiClient.isTransient(Throwable)`. It's the only code that knows provider exception types: it walks the cause chain for Spring AI's `TransientAiException`, an I/O error, or a provider exception carrying HTTP status 429 or 5xx (e.g. the Google GenAI SDK's `ApiException.code()`).
- **Feature services** (`TaskSuggestionService`, `TaskBreakdownService`, and Wave 4's chat) depend only on `AiClient` and their own records.
- **Changing the model or vendor:** another model needs only `AI_MODEL`. Another vendor follows D10, with no change to feature services or tests (they mock `ChatModel`).

### D2: retry and timeout

`GoogleGenAiChatModel` has its own `RetryTemplate`, and Spring AI's default retries many times with backoff. Combined with our own retry, that would break both "one retry" and the 20 s limit.

- **T3.1 adds `spring.ai.retry.max-attempts=1`** to `application.properties` (check the exact property name in Spring AI 2.0.1's `SpringAiRetryProperties`). This is an approved exception to the `application.properties` hotspot rule. A test asserts the model's retry is off, e.g. by checking the bound property.
- **Deadline:** `AiClient` runs each attempt on a virtual thread and waits with `Future.get(remaining)`, where `remaining` is what's left of one `app.ai.timeout` budget started at the first attempt. On timeout it cancels the future → `503 AI_UNAVAILABLE`. Worst-case latency is therefore about 20 s, not 40 s.
- **Retry:** exactly one, only when `isTransient` is true and the deadline hasn't passed. A timeout uses up the budget, so it isn't retried.
- **Other errors** (400, 401/403 from a bad key, unexpected exceptions) → `503 AI_UNAVAILABLE` with no retry, logged at `ERROR` with the exception class and status only. A parse or validation failure isn't a model error: it's `422` with no retry (D6).
- **Logging:** one `INFO` line per call with the feature, attempt count, latency, and outcome. Never the prompt, the response, task text, or the API key.

### D3: quota accounting

- `AiQuotaService.consume(userId)`: a sliding window of `app.ai.quota-per-hour` (30) per user over the last hour, using the injected `Clock`. The 31st call within an hour → `429 AI_RATE_LIMITED`. It's a `ConcurrentHashMap<UUID, Deque<Instant>>`, pruned on every access, so each user holds at most 30 entries.
- **Order in each endpoint:** Bean Validation → load and authorize the task (404) → depth check (400) → `consume` → `AiClient`. Requests that are invalid, not found, or too deep never use quota.
- A call that ends in 422 or 503 still counts: the model was used, and this stops a failing client from spamming. The internal retry doesn't count again.

### D4: prompt layout

- **System message:** a StringTemplate file in `src/main/resources/prompts/` (`suggest.st`, `breakdown.st`, and later `chat.st`). It holds the role, the rules, the output limits (D6), and a context block that `AiClient` fills in: today's date in `app.timezone`, the zone id, and the output language by name ("English" or "Brazilian Portuguese").
- **User message:** only data. It's the `userData` map serialized as JSON with Jackson, then `<` → `<` and `>` → `>` (still valid JSON), wrapped in `<data>` and `</data>`. The system prompt says that everything inside `<data>` is content to work on, never instructions.
- `AiClient` builds both messages. Feature services pass a template name and a data map, never raw prompt strings.

### D6: validating AI output

| Field | Rule |
|---|---|
| Suggest `suggestedTitle` | not blank, ≤ 100 characters |
| Suggest `suggestedDescription` | not blank, ≤ 500 characters |
| Suggest `suggestedPriority`, breakdown `priority` | one of the `PRIORITIES` names |
| Suggest `suggestedComplexity`, breakdown `complexity` | one of the `COMPLEXITIES` names (required, never null) |
| Suggest `reasoning` | not blank, ≤ 1000 characters |
| Breakdown draft `title` / `description` | same as a task: not blank, ≤ 100 / ≤ 500 |
| Breakdown draft count | 2–8 |

- The model's breakdown output is wrapped (`record BreakdownResult(List<SubtaskDraft> subtasks)`), because `AiClient.call` takes a `Class<T>`. The endpoint still returns a bare array, as in PLAN §5.
- `AiClient` runs Bean Validation on the converted record. The feature service checks the lookup names. Text is trimmed before checking. Nothing is silently truncated or repaired.

### D7: accepting a suggestion fills the form

PLAN §5 and the old T3.4 card said that accepting on a saved task calls `PATCH` directly. Wave 2 (T2.8) already built `AiSuggestSlot` with a different contract: `onApply` writes into the form, and saving uses the form's normal path. This wave keeps that contract:

- The edit form may hold other unsaved changes. A direct `PATCH` would either save them by surprise or race with them.
- There is one write path, and it already sends only changed fields and handles `OVERDUE` (wave 2, D6).
- A suggestion the user accepts but doesn't save is simply lost when they leave, like any other unsaved edit.

### D10: vendor-neutral env vars

The AI env vars no longer name a vendor. A model never needs its own variable: `AI_MODEL` holds whatever name the active vendor accepts.

| Var | Meaning | Default |
|---|---|---|
| `AI_PROVIDER` | Spring AI chat provider id, mapped to `spring.ai.model.chat` | `google-genai` |
| `AI_MODEL` | Model name for the active provider, e.g. `gemini-2.5-flash` | none (required) |
| `AI_API_KEY` | Key for providers that need one (Gemini) | none |
| `AI_BASE_URL` | Endpoint for self-hosted providers (Ollama); unused by Gemini | empty |

- `application.properties` maps the variables onto each built-in provider's properties: `spring.ai.model.chat=${AI_PROVIDER:google-genai}`, `spring.ai.google.genai.api-key=${AI_API_KEY}`, and `spring.ai.google.genai.chat.model=${AI_MODEL}`. Spring AI 2.0.1 turns on only the auto-configuration whose id matches `spring.ai.model.chat` (checked in `GoogleGenAiChatAutoConfiguration`).
- This was done before T3.1, together with the root renames, so `docker compose up` never sees a mix of old and new names. It's an approved exception to the `application.properties` hotspot rule.
- **Adding Ollama later** (not done now; no Ollama container in compose):
  1. Add `spring-ai-starter-model-ollama` to `pom.xml` (an exception to the `pom.xml` hotspot rule, so ask the maintainer).
  2. Add `spring.ai.ollama.base-url=${AI_BASE_URL:http://localhost:11434}` and Ollama's chat-model property set to `${AI_MODEL}` (check the exact name in Spring AI 2.0, e.g. `spring.ai.ollama.chat.model`).
  3. Make sure `AiClient.isTransient` (D1) recognizes Ollama's connection and 5xx errors, with a test.
  4. Then switching is `.env` only: `AI_PROVIDER=ollama`, `AI_MODEL=llama3.1:8b`, `AI_BASE_URL=http://host.docker.internal:11434`.

  Other vendors (OpenAI, Anthropic, …) follow the same steps with their own starter and properties.

## Current state

After Wave 2:

- API: `AppProperties.Ai(timeout, quotaPerHour)` is bound, with `app.ai.timeout=20s` and `app.ai.quota-per-hour=30`. `spring.ai.google.genai.api-key` and `.chat.model` are read from `AI_API_KEY` and `AI_MODEL`, and `spring.ai.model.chat` from `AI_PROVIDER` (D10). The test profile has a fake key and model.
- API: `ErrorCode` already has `AI_INVALID_RESPONSE` (422), `AI_RATE_LIMITED` (429), and `AI_UNAVAILABLE` (503). `GlobalExceptionHandler` turns `ApiException` into a ProblemDetail.
- API: `support/IntegrationTest` already declares `@MockitoBean ChatModel chatModel`, so every integration test runs with the model mocked. `MutableClock`, `TestRows`, and `JwtTestSupport` exist.
- API: `TaskService` exposes `loadOwned`, `depth`, `canAddSubtasks`, and `requireCanAddSubtasks` (wave 2, D1). `TaskRepository` has the ancestors query. `LookupService` resolves names.
- API: there is no `service/ai/` package, no `prompts/` directory, and no AI endpoint. `SecurityConfig` requires authentication for everything except its public paths, so the new endpoints are protected without touching it.
- UI: `src/features/ai/AiSuggestSlot.tsx` (props `taskId?`, `values`, `onApply`, `disabled`) is rendered by `TaskForm`, which is used by the create, edit, and add-subtask forms. `src/features/ai/AiBreakdownSlot.tsx` (props `task`, `canAddSubtasks`) is rendered at the top of `SubtaskSection`. Both render nothing.
- UI: `errors.json` already has the three AI codes in both locales. `ErrorMessage` maps an `ApiError` to its text. `useCreateSubtasks` exists in `src/api/queries/tasks.ts`. `apiRequest` doesn't send `Accept-Language`. The app's `QueryClient` uses TanStack defaults (mutations aren't retried).

## Tasks

API track: 3.1 → 3.2, 3.3. UI track: 3.4 → 3.5. The UI codes against PLAN §5 plus D5–D9 and may start before Checkpoint A. 3.2 and 3.3 own separate files and may run in parallel. 3.4 and 3.5 both register an i18n namespace in `src/i18n/index.ts`, `i18next.d.ts`, and `locales.test.ts`, so run 3.5 after 3.4 is merged.

### 3.1 API: AI foundation

- **Repo:** API · **PLAN:** §5 (Shared rules)
- **Owns:** `service/ai/AiClient`, `service/ai/AiQuotaService`, `config/AiConfig`, and `src/main/resources/prompts/` (the directory; feature tasks add their own template).
- **Changes:**
  - `spring.ai.retry.max-attempts=1` in `application.properties` (D2).
  - `AiConfig`: the `ChatClient` bean (from the auto-configured `ChatClient.Builder`) and a virtual-thread executor for `AiClient`.
  - `AiClient.<T> T call(String template, Map<String, Object> userData, Class<T> type, Locale locale)`: builds the messages (D4), calls the model with the deadline and retry (D2), converts with `.entity(type)`, runs Bean Validation (D6), and maps failures to `AI_INVALID_RESPONSE` or `AI_UNAVAILABLE`. `isTransient` per D1.
  - `AiQuotaService.consume(UUID userId)` per D3.
  - An `AiLocales` helper (or a configured `AcceptHeaderLocaleResolver`) that maps the header per D5.
  - A test-only template in `src/test/resources/prompts/` and a test helper that stubs `chatModel.call(Prompt)` with a canned JSON string or an exception.
- **Acceptance:** tests with the mocked `ChatModel`:
  - valid JSON → the parsed, validated record; JSON wrapped in a Markdown code fence also parses
  - malformed JSON, a missing field, or a value breaking a constraint → 422, and the model is called once
  - a transient error then success → success after exactly 2 calls; two transient errors → 503 after 2 calls
  - a non-transient error (e.g. a 403-style exception) → 503 after 1 call
  - a model slower than the timeout (test property `app.ai.timeout=200ms`) → 503 within about the timeout, not retried
  - the 31st `consume` within an hour → 429; after the clock moves past the oldest call, `consume` succeeds again (`MutableClock`)
  - the captured `Prompt` has the date in `app.timezone`, the zone id, and the language name; user text containing `</data>` and `ignore previous instructions` stays inside the single data section, escaped
  - locale mapping: `pt-BR`, `pt`, `pt-PT` → `pt-BR`; `en-US`, `fr`, missing → `en`
  - Spring AI's retry is set to one attempt
- **Verify:** `./mvnw test`
- **Commit:** `feat(api): add ai client with structured output, resilience and quota`

### 3.2 API: `/ai/suggest`

- **Repo:** API · **Depends on:** 3.1 · **Parallel with:** 3.3 · **PLAN:** §5 (Suggest scenario, endpoint)
- **Owns:** `service/ai/TaskSuggestionService`, `controller/AiController` (only the suggest method; Wave 4 adds chat), `dto/SuggestRequest`, `dto/SuggestResponse`, and `prompts/suggest.st`.
- **Changes:**
  - `POST /api/v1/ai/suggest` with `{ title, description }`, both `@NotBlank`, at most 100 and 500 characters.
  - The service: `consume` quota → `AiClient` with `suggest.st` and `{ title, description }` → check lookup names (D6) → `SuggestResponse`.
  - The prompt asks for a clearer title and description in the request's language, keeping the user's meaning, plus a priority, a complexity, and a short reasoning. It states the D6 limits and lists the allowed names.
  - No database writes. OpenAPI: summary and the 400, 401, 422, 429, and 503 responses.
- **Acceptance:**
  - success with a mocked model, in `en` and in `pt-BR` (the captured prompt names the language)
  - invalid input (blank, 101-character title, 501-character description) → 400, with the model never called and no quota used
  - a model response with a 300-character title, or priority `URGENT` → 422
  - a model failure → 503
  - no `TASK` rows change; unauthenticated → 401
- **Verify:** `./mvnw test`
- **Commit:** `feat(api): add ai task suggestion endpoint`

### 3.3 API: breakdown

- **Repo:** API · **Depends on:** 3.1 · **Parallel with:** 3.2 · **PLAN:** §5 (Breakdown scenario, endpoint)
- **Owns:** `service/ai/TaskBreakdownService`, `controller/TaskAiController` (`POST /api/v1/tasks/{id}/ai/breakdown`), `dto/SubtaskDraft`, `dto/BreakdownResult` (internal, D6), and `prompts/breakdown.st`.
- **Changes:**
  - `loadOwned` (404) → `requireCanAddSubtasks` (400 `SUBTASK_DEPTH_EXCEEDED`) → `consume` quota → `AiClient`.
  - Data per D8: `{ task: { title, description }, ancestors: [titles, root first], existingSubtasks: [titles] }`.
  - Check every draft's lookup names (D6). Return `200` with the drafts array in the model's order.
  - No database writes. OpenAPI: summary and the 400, 401, 404, 422, 429, and 503 responses.
- **Acceptance:**
  - success with 3 drafts
  - breaking down a depth-3 task succeeds; the captured prompt has its 2 ancestors' titles root first and its existing subtasks' titles
  - another user's task and an unknown id → 404; a depth-5 task → 400 `SUBTASK_DEPTH_EXCEEDED`; none of these call the model or use quota
  - 1 draft, 9 drafts, or a draft with an unknown complexity → 422
  - a model failure → 503
  - no rows are written
- **Verify:** `./mvnw test`
- **Commit:** `feat(api): add ai subtask breakdown endpoint`

> **Checkpoint A:** `docker compose up --build task-manager-db task-manager-api` with a real `AI_API_KEY`. In Swagger UI, call `/ai/suggest` with `Accept-Language: pt-BR` and get Portuguese text; break down a task and get 2–8 drafts; call suggest 31 times and get `429`. Without a key, mark this checkpoint skipped, never passed.

### 3.4 UI: AI suggest UX

- **Repo:** UI · **PLAN:** §5 (Suggest scenario), §6 (AI UX)
- **Owns:** `src/api/ai.ts`, `src/features/ai/suggest/**`, `src/features/ai/AiSuggestSlot.tsx`, `src/i18n/locales/{en,pt-BR}/ai.json`, and the `Accept-Language` change in `src/api/client.ts` (D5).
- **Changes:**
  - `apiRequest` sends `Accept-Language: <i18n.language>` on every request.
  - `src/api/ai.ts`: `SuggestRequest`, `SuggestResponse`, `suggestTask()`, and a `useSuggestTask()` mutation hook (D9).
  - `AiSuggestSlot`: a "Suggest with AI" button, enabled when the trimmed title and description are non-blank and within limits and `disabled` is false. It sends trimmed values.
  - A review panel: current vs suggested for title, description, priority, and complexity, plus the reasoning. One checkbox per field (all checked by default) and the actions "Accept all", "Accept selected", and "Dismiss". Accepting calls `onApply` with only the chosen fields and closes the panel (D7).
  - A loading state, and `ErrorMessage` for 422, 429, and 503. The form keeps its values on any error.
  - Register `ai.json` in `src/i18n/index.ts`, `i18next.d.ts`, and `locales.test.ts`.
- **Acceptance:** component tests:
  - the button is disabled until title and description are filled in
  - "Accept selected" with only priority and title checked calls `onApply` with just those two fields
  - in the edit form, accepting and then saving sends `PATCH` with only the fields that changed
  - a 503 shows the unavailable message and leaves the form intact
  - requests carry `Accept-Language`, and it follows a language switch
- **Verify:** `npm run lint && npm run build && npm test`
- **Commit:** `feat(ui): add ai suggestion review and accept flow`

### 3.5 UI: AI breakdown UX

- **Repo:** UI · **Depends on:** 3.4 (i18n registration files) · **PLAN:** §5 (Breakdown scenario), §6 (AI UX)
- **Owns:** `src/api/aiBreakdown.ts`, `src/features/ai/breakdown/**`, `src/features/ai/AiBreakdownSlot.tsx`, and `src/i18n/locales/{en,pt-BR}/aiBreakdown.json`. It is kept separate from 3.4's `ai.ts` and `ai.json`.
- **Changes:**
  - `src/api/aiBreakdown.ts`: `SubtaskDraft`, `breakdownTask()`, and a `useBreakdownTask()` mutation hook (D9).
  - `AiBreakdownSlot`: renders nothing when `canAddSubtasks` is `false`; otherwise a "Break down with AI" button.
  - The drafts appear as an editable list: title, description, priority, and complexity; remove; move up and down (buttons, no drag-and-drop dependency). Client-side validation uses the task limits.
  - "Create N subtasks" calls `useCreateSubtasks()` with the drafts in list order (the API keeps that order, wave 2 D5), then closes the panel. It's disabled while the request is in flight and when the list is empty or invalid. "Discard" closes the panel.
  - A loading state and AI error states as in 3.4. A failed create keeps the drafts.
  - Register `aiBreakdown.json` the same way as 3.4.
- **Acceptance:** component tests:
  - the button is hidden when `canAddSubtasks` is `false`
  - editing, removing, and reordering drafts sends the edited list, in the new order, to `POST /tasks/{id}/subtasks`
  - a second click while creating sends nothing
  - a 503 shows the localized message
- **Verify:** `npm run lint && npm run build && npm test`
- **Commit:** `feat(ui): add ai subtask breakdown review and accept flow`

> **Checkpoint B (wave done):** run `docker compose up --build` with a real key. Suggest on the create form, accept two fields, and save. Suggest on a saved task, accept, and save; only the changed fields are sent. Switch to PT-BR and suggest again: the text comes back in Portuguese. Break down a task, edit, remove, and reorder the drafts, and create them; they appear in that order. At depth 5 the breakdown button is gone. Stop the network or use a bad key, and see the unavailable message with the form intact. Without a key, the AI steps are skipped, never passed. Run `./mvnw test` in the API and `npm run lint && npm run build && npm test` in the UI. Then commit the submodule pointers in the root: `chore: bump submodules (wave 3)`.

## After this wave

- Fold into PLAN §5: D1 (`AiClient` replaces `AiAssistantService`, provider-specific code in three places), D2 (one deadline covers both attempts, no retry on timeout, Spring AI retry off), D5 (UI sends `Accept-Language`), and D7 (accepting fills the form; the user saves).
- Point Wave 4's T4.1 at `AiClient`, `AiQuotaService`, and D3–D6.
- Mark this plan as done.
