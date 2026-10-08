# Wave 4: Chat assistant

**Status:** in progress (2026-10-07); D1–D9 confirmed. T4.1 and T4.2 are merged into the submodules' `main`, and `./mvnw test` and the UI checks pass. Checkpoints A and B haven't been run yet · **PLAN:** §0 (Chat), §2 (Errors: 400, 422, 429, 503), §5 (shared rules, Chat scenario and endpoint), §6 (chat side panel), §8, §9 phase 4

## Goal

A logged-in user opens a side panel on any authenticated page and asks about their tasks in natural language ("Do I have overdue tasks?", "What should I do first?"). The API puts up to 20 of the user's open tasks in the prompt, most urgent first, and the AI answers only from them, in the user's language. The assistant is read-only: when asked to create or edit a task, it explains how to do that in the UI. Nothing is stored. The conversation lives in the panel until logout. Every call goes through wave 3's `AiClient`, so it is time-limited, retried at most once, and counted against the hourly quota, and every failure becomes `422`, `429`, or `503` with a localized message and the user's text still in the input. Tests never reach Gemini.

## Out of scope

- Tool calling: creating, editing, or completing tasks through chat (PLAN §9, Later).
- Persisting conversations, server-side sessions, or chat history across logins or tabs.
- Streaming replies, Markdown rendering, and a per-request choice of model.
- Search or retrieval beyond the 20 most urgent open tasks (no embeddings, no paging through tasks).
- Fixing wave 3's open latency issue (see D9). This wave measures it again.
- Tests against the real Gemini API. The manual checkpoints use a real key only when one is available.

## Decisions for this wave

Confirmed by the maintainer on 2026-10-07. D1–D6 of wave 3 (`AiClient`, retry and timeout, quota, prompt layout, locale, output validation) apply unchanged. Details below the table.

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

## After this wave

- Fold into PLAN §5: D1 (DTO names), D2–D3 (history item and reply limits), D4 (the full context order with tie-breakers, `openTaskCount`, and `parentTitle`), and D8 (a message joins the history only after a reply).
- Add the D7 router exception to the hotspot table in `TASKS.md`.
- Record Checkpoint A's latency numbers here and, if timeouts persist, raise D9 with the maintainer.
- Mark this plan as done.
