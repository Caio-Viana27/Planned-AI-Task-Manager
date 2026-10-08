# Wave 4: Chat assistant

**Status:** done (2026-10-07); D1–D9 confirmed, and D1–D4 and D8 folded into PLAN §5, D7 into the `TASKS.md` hotspot table. T4.1 and T4.2 committed, `./mvnw test` and the UI checks pass. Checkpoints A and B passed (2026-10-07, maintainer, real key); Checkpoint A's latency numbers weren't recorded here. Plus addendum D10–D14 (T4.3–T4.5): AI task analysis and estimated hours, T4.3–T4.5 committed 2026-10-08, `./mvnw test` (275) and the UI checks (331) pass; Checkpoint C pending (needs the maintainer and a real key) · **PLAN:** §0 (Chat), §2 (Errors: 400, 422, 429, 503), §5 (shared rules, Chat scenario and endpoint), §6 (chat side panel), §8, §9 phase 4

## Goal

A logged-in user opens a side panel on any authenticated page and asks about their tasks in natural language ("Do I have overdue tasks?", "What should I do first?"). The API puts up to 20 of the user's open tasks in the prompt, most urgent first, and the AI answers only from them, in the user's language. The assistant is read-only: when asked to create or edit a task, it explains how to do that in the UI. Nothing is stored. The conversation lives in the panel until logout. Every call goes through wave 3's `AiClient`, so it is time-limited, retried at most once, and counted against the hourly quota, and every failure becomes `422`, `429`, or `503` with a localized message and the user's text still in the input. Tests never reach Gemini.

**Addendum (D10–D14):** on a saved task's detail page, an "Analyze with AI" button between Edit and Delete asks the AI to review the existing task and suggest a priority, a complexity, an estimated effort in hours, and a short reason, in the user's language:

```json
{ "priority": "HIGH", "complexity": "MEDIUM", "estimatedHours": 8,
  "reason": "A tarefa envolve autenticação, controle de acesso e integração com o banco de dados." }
```

Estimated hours become a stored, editable task field. Accepting the analysis fills the edit form with the chosen fields; nothing is saved until the user clicks Save (PLAN §5, "Accepting suggestions").

## Out of scope

- Tool calling: creating, editing, or completing tasks through chat (PLAN §9, Later).
- Persisting conversations, server-side sessions, or chat history across logins or tabs.
- Streaming replies, Markdown rendering, and a per-request choice of model.
- Search or retrieval beyond the 20 most urgent open tasks (no embeddings, no paging through tasks).
- Fixing wave 3's open latency issue (see D9). This wave measures it again.
- Tests against the real Gemini API. The manual checkpoints use a real key only when one is available.
- Addendum: analysis from the dashboard row menu or the create form; storing the analysis `reason`; estimated hours on subtask creation (`POST /tasks/{id}/subtasks`) and in breakdown drafts; showing or sorting by hours in the dashboard list; adding hours to the chat context.

## Decisions for this wave

D1–D9 confirmed by the maintainer on 2026-10-07; D10–D14 added by the maintainer on 2026-10-08, after the wave. D1–D6 of wave 3 (`AiClient`, retry and timeout, quota, prompt layout, locale, output validation) apply unchanged. Details below the table.

| # | Decision | Outcome |
|---|---|---|
| D1 | DTO names | `dto/AiChatRequest`, `dto/AiChatMessage`, and `dto/AiChatResponse`, not `ChatRequest`/`ChatResponse`, which clash with Spring AI's `org.springframework.ai.chat.model.ChatResponse` in the same files. |
| D2 | Request limits | `message`: not blank, ≤ 1000 characters (PLAN §5). `history`: optional, ≤ 10 items; each item has `role` `user` or `assistant` and `content` not blank, ≤ 2000 characters (room for a full assistant reply, D3). Any breach → `400 VALIDATION_ERROR`, with no quota used. |
| D3 | Reply limits | `AiChatResponse(reply)`: not blank, ≤ 2000 characters, stated in the prompt. Plain text, no Markdown. Anything else → `422 AI_INVALID_RESPONSE` (wave 3, D6). |
| D4 | Chat context | Up to 20 of the caller's tasks whose status isn't `DONE`, at any depth, ordered `OVERDUE` first, then `dueDate` ascending (nulls last), then priority descending (by ID, wave 2 D3), then wave 2's tie-breakers `createdAt` desc and `id` asc. Plus `openTaskCount`, the total number of such tasks, so the AI can say it sees only the 20 most urgent. See below. |
| D5 | Prompt data | Everything goes in the `data` map, never as extra chat messages: `{ message, history, tasks, openTaskCount }`. Past assistant replies are data too, so a forged "assistant" turn can't give instructions. See below. |
| D6 | Read-only behavior | `chat.st` says the assistant can't change anything, answers only from `<data>`, says so plainly when the tasks don't hold the answer, and for create, edit, delete, or complete requests explains the UI steps (New task, the task's edit form, the status control, Delete, "Break down with AI"). |
| D7 | Panel placement | T4.2 passes `chatPanel={<ChatPanel />}` to the **protected** `AppLayout` in `src/routes/router.tsx`: a one-line, approved exception to the router hotspot rule. Guest pages get no panel. Both protected routes share that one layout element, so the conversation survives navigation between them. |
| D8 | Conversation state | Messages live in `ChatPanel` state only. A message is added to the list only after the reply arrives: on success the user message and the reply are appended together; on failure nothing is appended and the input keeps its text. Each request sends the last 10 list items as `history`. Logout unmounts the protected layout, which clears the chat; a test pins it. |
| D9 | Latency | No timeout change in this wave: `app.ai.timeout` stays 20 s. The chat prompt is larger than suggest's, so Checkpoint A records the latency of at least 5 real calls. If timeouts stay frequent, raising `app.ai.timeout` or picking a faster `AI_MODEL` is a separate maintainer decision. |
| D10 | Estimated hours field (addendum) | A new migration (the next free `V{n}`) adds `TASK.ESTIMATED_HOURS INT NULL` with `CHECK (ESTIMATED_HOURS BETWEEN 1 AND 999)`. The Task object gains `estimatedHours: integer \| null`. Optional on `POST /tasks`, `PUT`, and `PATCH` (in `PATCH`, absent = unchanged and `null` clears it, like `complexity`). Out of range or not an integer → `400 VALIDATION_ERROR`. Not on subtask create. |
| D11 | Analysis endpoint (addendum) | `POST /api/v1/tasks/{id}/ai/analysis`, no body, with `Accept-Language` → `200 { priority, complexity, estimatedHours, reason }`, in `TaskAiController` beside breakdown. Never writes. The ownership check (`404 TASK_NOT_FOUND`) runs before any quota is used (wave 3, D3). Works on a task in any status. |
| D12 | Analysis prompt data (addendum) | `{ task: { title, description, status, dueDate, priority, complexity, estimatedHours }, ancestors: [titles, root first], subtasks: [{ title, status }] }`, direct subtasks only. The current values are there so the AI can keep or change them; `analysis.st` says so and treats `<data>` as data only. |
| D13 | Analysis output (addendum) | `dto/TaskAnalysisResponse`: `priority` and `complexity` not blank and among the lookup names (complexity always set), `estimatedHours` an integer 1–999, `reason` not blank, ≤ 1000 characters, plain text, in the request language. Anything else → `422 AI_INVALID_RESPONSE` (wave 3, D6). |
| D14 | Analysis UI flow (addendum) | "Analyze with AI" sits between Edit and Delete and is hidden while editing, like them. The result shows under the header: current vs suggested priority, complexity, and hours, each with a checkbox (all checked), the reason, and "Apply to form" / "Dismiss". Applying opens the edit form with the task's values merged with the checked fields; Save sends only the changed fields via `PATCH` (`diffTaskForm`). Errors use `ErrorMessage` and leave the page unchanged. New i18n namespace `aiAnalysis.json`. |

### D4: chat context

- **Query** (owned by T4.1, in `TaskRepository`): JPQL with `JOIN FETCH` for `status` and `priority`, `LEFT JOIN FETCH` for `complexity` and `parent`, `WHERE t.user.id = :userId AND t.status <> :done`, and

  ```
  ORDER BY CASE WHEN t.status = :overdue THEN 0 ELSE 1 END,
           t.dueDate ASC NULLS LAST,
           t.priority.id DESC,
           t.createdAt DESC,
           t.id ASC
  ```

  with a `Limit.of(20)` (or `Pageable.ofSize(20)`). Status entities come from `LookupService`, never hard-coded IDs. A second query counts the same rows for `openTaskCount`.
- **Each task in the prompt:** `title`, `description`, `status`, `priority`, `complexity` (or `null`), `dueDate` (ISO date or `null`), and `parentTitle` (or `null` for a top-level task). No ids: the reply can't link to tasks, and ids only cost tokens.
- The 20 is a constant in the service, not a property: PLAN §5 fixes it.

### D5: prompt data

```json
{
  "message": "Do I have overdue tasks?",
  "history": [{ "role": "user", "content": "…" }, { "role": "assistant", "content": "…" }],
  "openTaskCount": 25,
  "tasks": [{ "title": "…", "description": "…", "status": "OVERDUE", "priority": "HIGH", "complexity": null, "dueDate": "2026-10-01", "parentTitle": null }]
}
```

- `AiClient` already escapes `<` and `>` and wraps the JSON in `<data>…</data>` (wave 3, D4), so nothing in a task, message, or history item can close the data section.
- Message and history text is trimmed before it goes in. Task text goes in as stored.
- The system prompt names `message` as the question to answer, `history` as earlier turns for context only, and `tasks` as the only source of facts.

## Current state

After Wave 3:

- API: `service/ai/AiClient.call(template, data, type, locale)` builds the messages, enforces the 20 s deadline and one retry, converts with `.entity(type)`, and runs Bean Validation. `AiQuotaService.consume(userId)` enforces 30 calls per user per hour. `AiLocales.fromAcceptLanguage` maps the header.
- API: `controller/AiController` (`/api/v1/ai`) has only `suggest`; its Javadoc already says chat comes later. `TaskSuggestionService` is the pattern to copy: `consume` → `AiClient` → extra checks.
- API: `prompts/` has `suggest.st` and `breakdown.st`. `AiClient` fills `{today}`, `{zone}`, and `{language}` in every template (and the lookup names).
- API: `TaskRepository` has no "open tasks for chat" query. `LookupService` resolves `DONE` and `OVERDUE`. `ErrorCode` already has every code this wave needs.
- API tests: `support/IntegrationTest` mocks `ChatModel`; `support/AiStubs` stubs canned JSON or an exception and captures the `Prompt`. `MutableClock` and `TestRows` exist.
- UI: `AppLayout` takes an optional `chatPanel` and renders it in a `w-80` `<aside>`, but `router.tsx` never passes one. `apiRequest` sends `Accept-Language`. `src/api/ai.ts` and `src/api/aiBreakdown.ts` show the mutation-hook pattern (wave 3, D9).
- UI: `errors.json` has `AI_INVALID_RESPONSE`, `AI_RATE_LIMITED`, `AI_UNAVAILABLE`, and `VALIDATION_ERROR` in both locales; `ErrorMessage` renders them. There is no `src/features/chat/` and no `chat.json`.

After T4.1 and T4.2 (for the addendum):

- API: `TaskBreakdownService` is the pattern for a task-scoped AI call: `taskService.loadOwned` → `quotaService.consume` → `AiClient.call` → lookup-name check, not transactional. `TaskAiController` hosts task-scoped AI endpoints. `TaskRepository.findAncestors` and `findChildren` give the D12 context. `PatchTaskRequest` tracks field presence, so a clearable field is one more constant left out of `@NotNullWhenPresent`.
- UI: `TaskDetailView` holds the Edit and Delete buttons and already mounts `TaskForm`, which takes `initialValues`. `taskToFormValues` and `diffTaskForm` live in `features/tasks/form/taskForm.ts`. `AiSuggestSlot` and `SuggestReview` show the mutation hook and the review-with-checkboxes pattern.

## Tasks

T4.1 (API) and T4.2 (UI) run in parallel. The UI codes against PLAN §5 plus D1–D3 and D7–D8, with mocked responses in tests. T4.2 registers `chat.json` in `src/i18n/index.ts`, `i18next.d.ts`, and `locales.test.ts`; Wave 5's T5.3 registers `passwordReset.json` in the same files, so T5.3 starts after T4.2 is merged.

### 4.1 API: chat

- **Repo:** API · **Depends on:** T3.1, T2.2 · **PLAN:** §0 (Chat), §5 (Chat scenario, endpoint)
- **Owns:** `service/ai/ChatAssistantService`, the `chat` method in `controller/AiController`, `dto/AiChatRequest`, `dto/AiChatMessage`, `dto/AiChatResponse` (D1), `prompts/chat.st`, and the two chat-context queries in `TaskRepository` (D4).
- **Changes:**
  - `POST /api/v1/ai/chat` with `@Valid AiChatRequest` (limits per D2) and the `Accept-Language` header, like `suggest`.
  - `ChatAssistantService.chat(request, locale)`: `consume` quota → load the context (D4) → `AiClient.call("chat", data, AiChatResponse.class, locale)` with the data map from D5 → return the reply. Don't use `ChatClient` or add retry or timeout code (wave 3, D1 and D2).
  - `chat.st` per D6, stating the reply limit (D3), with `{today}`, `{zone}`, and `{language}` in its context block, like `suggest.st`.
  - No database writes. OpenAPI: summary, a description that mentions the quota and the read-only behavior, and the 400, 401, 422, 429, and 503 responses.
- **Acceptance:** integration tests with the mocked `ChatModel` (`support/AiStubs`):
  - success returns `{ reply }`; in `pt-BR` the captured prompt names Brazilian Portuguese
  - with 25 open tasks for the caller, 3 `DONE` tasks, and 5 tasks of another user: the captured prompt has exactly 20 tasks, none of them `DONE` or another user's, in D4 order (an `OVERDUE` task first, then by due date with nulls last, then `HIGH` before `LOW` for equal dates), and `openTaskCount` is 25
  - a subtask in the context carries its `parentTitle`
  - a user with no open tasks gets a reply, and the prompt has an empty `tasks` list and `openTaskCount` 0
  - `history` items appear inside the single `<data>` section in order; a history item containing `</data>` stays escaped inside it
  - validation → 400 with the model never called and no quota used: blank message, a 1001-character message, 11 history items, a `system` role, a blank or 2001-character history item
  - a blank or 2001-character reply → 422
  - a model failure → 503
  - unauthenticated → 401; no `TASK` rows change
- **Verify:** `./mvnw test`
- **Commit:** `feat(api): add read-only chat assistant endpoint`

> **Checkpoint A:** `docker compose up --build task-manager-db task-manager-api` with a real `AI_API_KEY`. In Swagger UI, with a few tasks seeded (one overdue, one with a subtask), ask "Do I have overdue tasks?" and get an answer naming the overdue task; ask with `Accept-Language: pt-BR` and get Portuguese; ask it to create a task and get UI instructions, with no new task in the list. Record the latency of at least 5 calls (D9). Without a key, mark this checkpoint skipped, never passed.

### 4.2 UI: chat panel

- **Repo:** UI · **Depends on:** T1.6 (layout slot) · **PLAN:** §5 (Chat scenario), §6
- **Owns:** `src/api/chat.ts`, `src/features/chat/**`, `src/i18n/locales/{en,pt-BR}/chat.json`, and the one-line `chatPanel` prop in `src/routes/router.tsx` (D7).
- **Changes:**
  - `src/api/chat.ts`: `ChatMessage`, `ChatRequest`, `ChatResponse` types (these names are fine in the UI), `sendChatMessage()`, and a `useSendChatMessage()` mutation hook: never cached, never auto-retried (wave 3, D9).
  - `ChatPanel`, a collapsible panel in the `AppLayout` slot, collapsed by default, with a toggle button that has an accessible name. When open: the message list (user and assistant messages styled apart, text with `whitespace-pre-wrap`, no Markdown), an input with a 1000-character limit, and a send button.
  - Sending: trimmed, non-blank text only. `history` is the last 10 list items, mapped to `{ role, content }`. While waiting, show a typing indicator and disable sending (button and Enter). Conversation state per D8.
  - Errors: `ErrorMessage` for 400, 422, 429, and 503. The input keeps its text and nothing is added to the list.
  - An empty state that suggests example questions, and a "Clear chat" action.
  - Register `chat.json` in `src/i18n/index.ts`, `i18next.d.ts`, and `locales.test.ts`.
- **Acceptance:** component tests:
  - after 6 exchanges (12 messages), the next request's `history` holds exactly the last 10 messages, in order, with the right roles
  - a failed send (503, then 429) keeps the input text, shows the localized message, and adds nothing to the list
  - a second send while one is in flight sends nothing, and the typing indicator shows
  - a blank or whitespace-only message can't be sent
  - logging out and back in shows an empty chat; navigating from the dashboard to a task detail page keeps it
  - the panel isn't rendered on `/login`
- **Verify:** `npm run lint && npm run build && npm test`
- **Commit:** `feat(ui): add chat assistant panel`

> **Checkpoint B (wave done):** run `docker compose up --build` with a real key. Open the panel on the dashboard, ask about overdue tasks, then follow up ("and which one is most urgent?") and check the reply uses the earlier turn. Navigate to a task and back: the conversation is still there. Switch to PT-BR and ask again: the reply is in Portuguese. Ask it to create a task: it explains the UI steps and the task list doesn't change. Use a bad key or stop the network: the unavailable message shows and the input keeps its text. Log out and back in: the chat is empty. Without a key, the AI steps are skipped, never passed. Run `./mvnw test` in the API and `npm run lint && npm run build && npm test` in the UI. Then commit the submodule pointers in the root: `chore: bump submodules (wave 4)`.

### 4.3 API: estimated hours field (addendum, D10)

- **Repo:** API · **Depends on:** T2.2 · **PLAN:** §1, §2, §4
- **Changes:**
  - The D10 migration.
  - `Task.estimatedHours` (`Integer`) and `TaskResponse.estimatedHours`.
  - An optional `@Min(1) @Max(999) Integer estimatedHours` on `CreateTaskRequest`, `UpdateTaskRequest`, and `PatchTaskRequest` (a new `ESTIMATED_HOURS` constant, clearable, so not in `@NotNullWhenPresent`).
  - `TaskService` create, update, and patch map it. OpenAPI examples.
- **Acceptance:**
  - create with and without hours; `GET` returns it, `null` when unset
  - `PATCH` sets it, `PATCH` with `null` clears it, `PATCH` without it leaves it; `PUT` sets it
  - `0`, `1000`, and `2.5` → 400
  - the database check rejects an out-of-range value written directly
- **Verify:** `./mvnw test`
- **Commit:** `feat(api): add estimated hours to tasks`

### 4.4 API: task analysis (addendum, D11–D13)

- **Repo:** API · **Depends on:** T4.3, T3.1 · **PLAN:** §5
- **Owns:** `service/ai/TaskAnalysisService`, `dto/TaskAnalysisResponse`, `prompts/analysis.st`, and the `analysis` method in `controller/TaskAiController`.
- **Changes:**
  - `TaskAnalysisService.analyze(taskId, locale)`, following `TaskBreakdownService`: `taskService.loadOwned` → `quotaService.consume` → the D12 data map → `aiClient.call("analysis", data, TaskAnalysisResponse.class, locale)` → the lookup-name check. Not transactional, so no connection is held during the AI call. No retry or timeout code (wave 3, D1 and D2).
  - `analysis.st` like `suggest.st`: `{today}`, `{zone}`, `{language}`, `{priorities}`, and `{complexities}`, the D13 limits, and `<data>` as data only.
  - OpenAPI: summary, a description that mentions the quota and that nothing is saved, and the 401, 404, 422, 429, and 503 responses.
- **Acceptance:** integration tests with the mocked `ChatModel` (`support/AiStubs`):
  - success returns the four fields; in `pt-BR` the captured prompt names Brazilian Portuguese
  - the captured prompt holds the task's current values, its ancestor titles root first, and its direct subtasks
  - another user's task → 404, with the model never called and no quota used
  - an unknown priority, a null complexity, `estimatedHours` 0 or 1000, a blank or 1001-character reason → 422
  - a model failure → 503; the quota exhausted → 429; unauthenticated → 401
  - no `TASK` rows change
- **Verify:** `./mvnw test`
- **Commit:** `feat(api): add AI task analysis endpoint`

### 4.5 UI: estimated hours and "Analyze with AI" (addendum, D10, D14)

- **Repo:** UI · **Depends on:** T2.8, T3.4 · **PLAN:** §5, §6
- Codes against D10–D13 with mocked responses, so it can run in parallel with T4.3 and T4.4.
- **Owns:** `src/api/aiAnalysis.ts`, `src/features/ai/analysis/**`, and `src/i18n/locales/{en,pt-BR}/aiAnalysis.json`. Also edits `src/api/tasks.ts` (types), `features/tasks/form/{taskForm.ts,fields.tsx,TaskForm.tsx}`, `features/tasks/detail/{TaskDetailView.tsx,TaskFields.tsx}`, and `tasks.json` (the field label and validation messages).
- **Changes:**
  - `Task.estimatedHours: number | null` and the request types. `TaskFormValues.estimatedHours: string` (`''` = none), handled in `taskToFormValues`, `validateTaskForm` (an integer 1–999), `diffTaskForm` (`''` → `null`), and the create request.
  - A number input in the task form; `TaskFields` shows the hours, or "Not estimated".
  - `useAnalyzeTask()` mutation hook in `src/api/aiAnalysis.ts`: never cached, never auto-retried (wave 3, D9).
  - An analyze button and an `AnalysisReview` in `features/ai/analysis/`, wired into `TaskDetailView` per D14. `TaskDetailView` keeps the values to open the edit form with, so "Apply to form" opens `TaskForm` with the merged values.
  - Register `aiAnalysis.json` in `src/i18n/index.ts`, `i18next.d.ts`, and `locales.test.ts`.
- **Acceptance:** component tests:
  - the button sits between Edit and Delete, is hidden while editing, and is disabled while loading
  - the review shows current vs suggested values and the reason
  - applying with one field unchecked opens the form with only the checked fields changed, and Save sends a `PATCH` with exactly those fields
  - Dismiss hides the review; 422, 429, and 503 show the localized message and change nothing
  - the form rejects hours 0, 1000, and 2.5, and clearing the field sends `estimatedHours: null`
- **Verify:** `npm run lint && npm run build && npm test`
- **Commit:** `feat(ui): add estimated hours and AI task analysis`

> **Checkpoint C (addendum done):** run `docker compose up --build` with a real key. Open a task, click "Analyze with AI", uncheck one field, apply, and save: the task shows the new values and hours. Switch to PT-BR and analyze again: the reason is in Portuguese. Use a bad key: the unavailable message shows and the task is unchanged. Without a key, the AI steps are skipped, never passed. Run `./mvnw test` in the API and `npm run lint && npm run build && npm test` in the UI. Then commit the submodule pointers in the root: `chore: bump submodules (wave 4 addendum)`.

## After this wave

- Fold into PLAN §5: D1 (DTO names), D2–D3 (history item and reply limits), D4 (the full context order with tie-breakers, `openTaskCount`, and `parentTitle`), and D8 (a message joins the history only after a reply).
- Add the D7 router exception to the hotspot table in `TASKS.md`.
- Record Checkpoint A's latency numbers here and, if timeouts persist, raise D9 with the maintainer.
- Mark this plan as done.
- Addendum: fold D10 into PLAN §1 and §2 (Task object, Validation and defaults) and the §4 endpoint bodies; D11–D14 into PLAN §5 (an "Analyze a task" scenario and the endpoint row) and §6 (AI UX). Then mark the addendum done.
