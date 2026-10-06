# Wave 1: Auth

**Status:** planned, all decisions confirmed (2026-10-06) · **PLAN:** §0 (Logout, JWT, Roles, User name), §2 (Errors: 401, 409), §3 (except Phase 5), §6 (Auth), §9 phase 1

## Goal

A user can sign up, sign in, see their profile through `GET /api/v1/users/me`, and log out. Every endpoint except sign-up, sign-in, the healthcheck, and the API docs requires a valid JWT. In the UI, protected routes redirect to `/login`, an expired session sends the user back to `/login` with a message, and the header shows who is logged in.

## Out of scope

- Password reset (Wave 5). The "Forgot your password?" link on the login page stays, pointing at the placeholder page.
- Refresh tokens, a `/logout` endpoint, and admin endpoints (PLAN §0).
- Anything task-related. The dashboard stays a placeholder, but it is now behind login.
- Rate limiting sign-in attempts. Not in PLAN for v1.

## Decisions for this wave

Confirmed by the maintainer on 2026-10-06. D4, D5, and D7 are detailed below the table.

| # | Decision | Outcome |
|---|---|---|
| D1 | JWT dependency | Wave 0 didn't add the OAuth2 resource server starter. Task 1.2 adds it (the Boot 4 starter name, `spring-boot-starter-security-oauth2-resource-server`; check the Boot 4.1 docs). This is the one dependency Wave 1 adds, an approved exception to the `pom.xml` hotspot rule. |
| D2 | One role per user | V1 has `USERS.ROLE_ID`, a single role, not a join table. Keep it. `User` has a `@ManyToOne Role`. The `roles` claim and the `/users/me` `roles` field are still arrays (of one item), so a later many-to-many change won't break the contract. |
| D3 | Timestamp type | V1 uses `TIMESTAMP` (no time zone). Entities use `Instant`, set from the `Clock`, stored as UTC. Task 1.1 adds `spring.jpa.properties.hibernate.type.preferred_instant_jdbc_type=TIMESTAMP` and `spring.jpa.properties.hibernate.jdbc.time_zone=UTC` to `application.properties`, so `ddl-auto=validate` accepts the columns. Wave 2's `Task` follows the same rule. An approved exception to the `application.properties` hotspot rule. |
| D4 | Email case | A new migration adds a unique index on `LOWER(EMAIL)`, so the database rejects emails that differ only in case. The service also stores emails lowercase, and a race on sign-up returns 409. See below. |
| D5 | Field limits | Limits for every sign-up and sign-in field, with BCrypt's 72-byte limit handled. See below. |
| D6 | JWT secret encoding | `JWT_SECRET` is used as raw UTF-8 bytes, not base64-decoded. Startup fails if it's shorter than 32 bytes. The maintainer's secret is two concatenated random UUIDs: 64 to 72 bytes, depending on hyphens. The UUIDs must be random (v4), not time-based (v1), so the secret has about 244 random bits. |
| D7 | Session restore in the UI | The UI loads the user from `/users/me` and never decodes the JWT. See below. |

### D4: email case

**Problem.** `USERS.EMAIL` has a plain `UNIQUE` constraint, which is case-sensitive. On its own it lets `Ana@x.com` and `ana@x.com` become two accounts.

**Rule: the database guarantees it.** A new migration adds a case-insensitive unique index:

```sql
CREATE UNIQUE INDEX UX_USERS_EMAIL_LOWER ON USERS (LOWER(EMAIL));
```

- With it, no write path (sign-up, a future admin tool, a manual `INSERT`) can create two emails that differ only in letter case.
- The existing `UNIQUE (EMAIL)` stays. It's redundant for uniqueness, but it's the index that serves `WHERE EMAIL = ?` lookups.
- The migration can't fail on existing data: no environment has users yet, because sign-up doesn't exist before this wave.
- Following `RULES.md`, it is a new file with the next free version number; V1 is never edited.

**The service still normalizes,** so stored emails look consistent and lookups stay simple:

- `AuthService` lowercases with `email.toLowerCase(Locale.ROOT)` on sign-up and on sign-in. `Locale.ROOT` avoids locale surprises such as the Turkish dotless `i`.
- The API doesn't trim. `@Email` already rejects leading or trailing spaces with a `400`. The UI trims the email before sending, so users never hit that.
- Repository queries use the normalized value with `findByEmail` and `existsByEmail`, served by the existing unique index. No `IgnoreCase` queries.
- `/users/me` and `AuthResponse.user` return the stored, lowercase email.

**Race.** Sign-up checks `existsByEmail`, then saves. Two concurrent sign-ups with the same email can both pass the check. The second insert then fails on a unique constraint, which would be a `500`. To return `409` instead:

- Save with `saveAndFlush` inside the sign-up transaction, so the violation surfaces in `AuthService` and not at commit.
- Catch `DataIntegrityViolationException` and throw `EMAIL_ALREADY_USED` only when the violated constraint is one of the two email ones (`users_email_key` or `ux_users_email_lower`, read from Hibernate's `ConstraintViolationException.getConstraintName()`). Any other integrity error is a bug and keeps propagating as a `500`.

**Tests.**

- Migration (1.1): inserting `Ana@x.com` and then `ana@x.com` with `JdbcTemplate` fails on `ux_users_email_lower`.
- Race (1.4): a `@MockitoSpyBean UserRepository` whose `existsByEmail` returns `false` for an email already in the table. Sign-up returns `409 EMAIL_ALREADY_USED`.
- Unnormalized row (1.4): a row with `Ana@x.com` inserted directly with `JdbcTemplate`, bypassing the service. Sign-up with `ana@x.com` passes the `existsByEmail` check, hits `ux_users_email_lower`, and returns `409`.

### D5: field limits

PLAN §3 only requires a password of at least 8 characters. These limits come from the V1 columns and from BCrypt. The orchestrator adds them to PLAN §3 after the wave.

**`SignUpRequest`:**

| Field | Constraints | Why |
|---|---|---|
| `email` | `@NotBlank`, `@Email`, `@Size(max = 100)` | `EMAIL VARCHAR(100)` |
| `name` | `@NotBlank`, `@Size(max = 100)` | `NAME VARCHAR(100)` and its `CHECK (NAME ~ '\S')`. The service strips leading and trailing spaces before saving. |
| `password` | `@NotNull`, `@Size(min = 8)`, `@MaxUtf8Bytes(72)` | See "BCrypt" below. Spaces are allowed and kept. No other complexity rules in v1. |

**`SignInRequest`:** `email` and `password` are only `@NotBlank`. Sign-in doesn't check formats or lengths, so a failed sign-in never reveals the sign-up rules. Anything that isn't a match is `401 BAD_CREDENTIALS`.

**BCrypt.** BCrypt uses only the first 72 **bytes** of a password. Spring Security's `BCryptPasswordEncoder` rejects longer input with an `IllegalArgumentException`, which would be a `500`. A `@Size(max = 72)` limit isn't enough, because it counts characters: 72 non-ASCII characters can be up to 288 UTF-8 bytes. So:

- Sign-up uses a small custom constraint, `dto/validation/MaxUtf8Bytes`, that counts UTF-8 bytes. Its message is "must be at most 72 bytes".
- Sign-in checks the byte length in `AuthService` first. A password over 72 bytes is treated as a wrong password: it gets the same dummy BCrypt check and the same `401 BAD_CREDENTIALS`, without calling the encoder on the long input.

**Error messages.** The API's `errors[].message` values are Bean Validation's default English messages. They help API users and Swagger, but the UI never shows them. The UI:

- runs the same checks client-side (counting password bytes with `TextEncoder`), with messages from `auth.json`
- maps a server field error to a localized message by field name (`auth.fieldErrors.email`, `.name`, `.password`), for the rare case where the server rejects something the client accepted

### D7: session restore in the UI

**On app load:**

1. No token in storage → `status = anonymous`. No request.
2. Token present → `status = loading`. A `['me']` query calls `GET /users/me` (`staleTime: Infinity`, `retry: false`).
3. `200` → `status = authenticated`, with the user.
4. `401` → the client's global handler logs out and navigates to `/login?expired=1`. An old token from a previous visit counts as an expired session, so the banner shows.
5. Network error or `5xx` → the user is **not** logged out, and the token stays. `ProtectedRoute` shows `errors.generic` with a "Try again" button that refetches `['me']`. A brief API outage mustn't end a valid session.

**Why not decode the JWT.** The server is the only judge of a token. Decoding `exp` client-side needs a library or hand-written base64url parsing, and it breaks if the browser's clock is wrong. The cost: a tab left open past the 60-minute TTL keeps showing its data until the next request, which then redirects. That's acceptable for v1.

**User shape.** `AuthResponse.user` is `{ id, name, email }`, but `/users/me` also returns `roles`. The session only holds `AuthUser { id, name, email }`, because v1 has no role-dependent UI. `getMe` maps its response to that type. `login(response)` can then seed `['me']` from `response.user` with no extra request.

**Logout order.** `logout()` clears the token, navigates to `/login` with `replace: true`, then calls `queryClient.clear()`. Navigating before clearing stops the protected page's queries from refetching without a token. The 401 handler does the same, with `?expired=1`. It's guarded so that several requests failing with `401` at once redirect only once.

**Other tabs.** `AuthProvider` listens to the `storage` event for the token key. When another tab logs out, this tab logs out too, without `?expired=1`. When another tab logs in, this tab picks up the token and refetches `['me']`. Without this, a tab whose token was removed elsewhere sends requests without a token. Those `401`s don't trigger the handler, so the tab would show errors and never redirect.

## Current state

After Wave 0:

- API: `config/SecurityConfig` is the Wave 0 stub. It permits the healthcheck and the docs, answers everything else with an empty `401` from `HttpStatusEntryPoint`, and defines an empty `InMemoryUserDetailsManager` to stop Boot from creating a default user. `ErrorCode` already has `UNAUTHORIZED`, `BAD_CREDENTIALS`, and `EMAIL_ALREADY_USED`. `AppProperties.Jwt` has `secret` and `ttl`. `OpenApiConfig` requires the bearer scheme on every operation by default; public operations opt out with an empty `@SecurityRequirements`.
- API: no entities, repositories, services, or controllers yet. There is no JWT library on the classpath (see D1).
- UI: `src/api/client.ts` already adds the `Authorization` header and calls a registered `onUnauthorized` handler on a `401` for requests that carried a token. `tokenStorage.ts` is done. `ProtectedRoute` is a stub that always renders. `AppLayout` always shows the "Log in" and "Sign up" links. `LoginPage` and `SignupPage` are placeholders, and their titles live in `common.json` (`pages.login`, `pages.signup`).
- UI: `errors.json` already has `UNAUTHORIZED` ("Your session has expired"), `BAD_CREDENTIALS`, and `EMAIL_ALREADY_USED` in both locales. `renderRoute` (the test helper) mounts the routes without a `QueryClientProvider`.

## Tasks

Run them in order, one at a time. Review and merge each before starting the next. The UI tasks code against the PLAN §3 contract, not the running API, so 1.5 may start before Checkpoint A if the API side is waiting on review.

### 1.1 API: user and role persistence

- **Repo:** API · **PLAN:** §0 (Roles, User name), §3
- **Changes:**
  - A new migration with the next free version (e.g. `…__users-email-case-insensitive-unique.sql`) that creates `UX_USERS_EMAIL_LOWER` (D4).
  - Add the two D3 Hibernate properties to `application.properties`. The test profile inherits them.
  - `entity/Role` (maps `ROLE`) and `entity/User` (maps `USERS`): UUID id generated by the app, `name`, `email`, `password` (the hash), `@ManyToOne(fetch = LAZY) Role role` (D2), and `createdAt`/`updatedAt` as `Instant`.
  - The service sets `createdAt` and `updatedAt` from the injected `Clock`, not from `@CreationTimestamp`, so tests can fix the time.
  - `repository/RoleRepository` with `findByName`.
  - `repository/UserRepository` with `findByEmail` and `existsByEmail`. Emails are stored lowercase (D4), so no `IgnoreCase` query is needed, and the existing `UNIQUE (EMAIL)` index serves them.
- **Acceptance:** repository tests (extending `IntegrationTest`) save a user with role `USER`, find it by email, and read back the role name and timestamps unchanged. Two emails differing only in case can't both be inserted (D4). The context starts with `ddl-auto=validate`. The seeded `USER` and `ADMIN` roles are found by name.
- **Commit:** `feat(api): add user and role entities and case-insensitive email uniqueness`

### 1.2 API: JWT issuing and decoding

- **Repo:** API · **PLAN:** §0 (JWT)
- **Changes:**
  - `pom.xml`: the OAuth2 resource server starter (D1).
  - `config/JwtConfig`:
    - a `SecretKey` (HmacSHA256) from `app.jwt.secret` (D6). Fail at startup with a clear message if it's shorter than 32 bytes.
    - `JwtEncoder` (`NimbusJwtEncoder`) and `JwtDecoder` (`NimbusJwtDecoder.withSecretKey(...).macAlgorithm(HS256)`). The decoder's `JwtTimestampValidator` uses the `Clock` bean, so tests can expire a token.
    - `PasswordEncoder`: `BCryptPasswordEncoder`.
  - `service/JwtService`: `issue(User)` returns the token and its `expiresAt` (now + `app.jwt.ttl`). Claims: `sub` = user id, `email`, `roles` (array, D2), `iat`, `exp`. Header `alg` = `HS256`.
- **Acceptance:** unit tests show a token round-trips through the decoder with the right claims; the decoder rejects a token past `exp` (fixed `Clock`), a token with a changed payload, and a token signed with another secret; startup fails with a 31-byte secret.
- **Review:** `pom.xml`. This is the only dependency change in Wave 1.
- **Commit:** `feat(api): add jwt issuing and decoding`

### 1.3 API: JWT security

- **Repo:** API · **PLAN:** §0 (Logout, JWT), §2 (401)
- **Changes:**
  - Replace the Wave 0 `config/SecurityConfig`:
    - stateless, CSRF off, CORS from `CorsConfig`, no HTTP Basic or form login
    - `oauth2ResourceServer().jwt()`, with a `JwtAuthenticationConverter` that turns the `roles` claim into `ROLE_*` authorities
    - public: `POST /api/v1/auth/signup`, `POST /api/v1/auth/signin`, `/actuator/health`, `/swagger-ui.html`, `/swagger-ui/**`, `/v3/api-docs/**`; everything else requires authentication
    - an `AuthenticationEntryPoint` and `AccessDeniedHandler` that write a `ProblemDetail` with code `UNAUTHORIZED` (`application/problem+json`), shaped like the ones `GlobalExceptionHandler` writes. No detail about why the token was rejected.
    - remove the empty `InMemoryUserDetailsManager`, unless Boot still creates a default user without it
  - `service/CurrentUserService`: returns the authenticated user's id (UUID from `sub`). Wave 2 uses it to scope every task query.
  - Update `SecurityConfigTest` for the new rules.
- **Acceptance:** tests show:
  - no token → 401 `UNAUTHORIZED` with a ProblemDetail body
  - expired token (fixed `Clock`) → 401
  - tampered token, wrong secret, and `alg: none` → 401
  - valid token → passes, and `CurrentUserService` returns the `sub`
  - the public paths still need no token, and a CORS preflight from `http://localhost:5173` still gets its headers
- **Commit:** `feat(api): add stateless jwt security`

### 1.4 API: auth and current-user endpoints

- **Repo:** API · **PLAN:** §3 (BDD and endpoints, except Phase 5)
- **Changes:**
  - `dto/` records, validated per D5: `SignUpRequest { email, name, password }`, `SignInRequest { email, password }`, `AuthResponse { token, expiresAt, user }`, `AuthUserResponse { id, name, email }`, `UserResponse { id, name, email, roles }`.
  - `service/AuthService`:
    - sign-up: lowercase the email and strip the name (D4, D5), check `existsByEmail`, hash the password, assign role `USER` by name, `saveAndFlush`, and issue a token. Map a violation of either email constraint to `EMAIL_ALREADY_USED` (D4).
    - sign-in: lowercase the email, load the user, check the password. An unknown email, a wrong password, and a password over 72 bytes all throw the same `BAD_CREDENTIALS` (D5). When there's no user to check against, still run one BCrypt check against a fixed dummy hash, so response time doesn't reveal which emails exist.
  - `dto/validation/MaxUtf8Bytes`: a constraint annotation and validator that counts UTF-8 bytes (D5).
  - `service/UserService` (or a method on `AuthService`): load the current user by id for `/users/me`.
  - `controller/AuthController`: `POST /api/v1/auth/signup` (201) and `POST /api/v1/auth/signin` (200), both with an empty `@SecurityRequirements`.
  - `controller/UserController`: `GET /api/v1/users/me`.
  - OpenAPI annotations: summaries, and the documented error responses for each operation.
- **Acceptance:** one integration test per §3 scenario, named after it:
  - `signUp_withValidData_returns201AndUsableToken`: the token works on `/users/me`, and `roles` is `["USER"]`
  - `signUp_withExistingEmail_returns409`, also with a different letter case
  - `signUp_whenEmailTakenConcurrently_returns409`, with the spied repository from D4
  - `signUp_whenMixedCaseRowExists_returns409`, with the row inserted directly (D4)
  - `signUp_storesLowercaseEmailAndStrippedName`
  - `signUp_withPasswordOver72Bytes_returns400`, using multi-byte characters that fit in 72 characters
  - `signUp_withInvalidFields_returns400`, with an `errors` entry for each bad field
  - `signUp_storesBcryptHash`: the stored password isn't the raw password and matches with `PasswordEncoder`
  - `signIn_withValidCredentials_returns200`, also with an upper-case email
  - `signIn_withWrongPassword_returns401`, `signIn_withUnknownEmail_returns401`, and `signIn_withPasswordOver72Bytes_returns401` produce the same body
  - `me_withoutToken_returns401`
- **Commit:** `feat(api): add sign-up, sign-in and current-user endpoints`

> **Checkpoint A:** `docker compose up --build task-manager-db task-manager-api`. In Swagger UI, sign up, sign in, click "Authorize" with the token, and call `/users/me`. Call it again without the token and get a 401 ProblemDetail. The healthcheck stays healthy.

### 1.5 UI: auth API and session

- **Repo:** UI · **PLAN:** §3, §6 (Auth)
- **Changes:**
  - `src/api/auth.ts`: `signUp`, `signIn`, and `getMe`, with the types `AuthResponse` and `AuthUser { id, name, email }`. `getMe` maps the `/users/me` response to `AuthUser`, leaving out `roles` (D7).
  - `src/auth/AuthProvider.tsx` and a `useAuth()` hook, following D7:
    - state: the token (initialized from `tokenStorage`) and the current user, loaded with a `['me']` query (`staleTime: Infinity`, `retry: false`) only when a token exists
    - `status`: `anonymous`, `loading`, `authenticated`, or `error` (network error or `5xx` on `['me']`; the token is kept), plus a `retry()` for the error case
    - `login(response)`: stores the token and seeds `['me']` from `response.user`
    - `logout()`: clears the token, navigates to `/login` with `replace: true`, then calls `queryClient.clear()`
    - registers the `setUnauthorizedHandler`: the same as `logout()`, but to `/login?expired=1`, and guarded so it runs once per session. It unregisters on unmount.
    - listens to the `storage` event for the token key, to follow logins and logouts in other tabs
  - The provider needs the router to navigate, so mount it as a root layout route in `src/routes/router.tsx` that wraps both existing route groups. This is an approved one-time edit to the router hotspot. `renderRoute` picks it up automatically.
  - `src/test/renderRoute.tsx`: wrap in a fresh `QueryClientProvider` per test (retries off), and allow an initial token.
- **Acceptance:** tests show:
  - `login` stores the token and exposes the user
  - with a stored token, the user comes from `/users/me`
  - `logout` clears the token and the query cache, and lands on `/login`
  - a `401` on an authenticated request logs out and lands on `/login?expired=1`, once, even when two requests fail together
  - a `500` from `/users/me` keeps the token and exposes `status = error`, and `retry()` recovers
  - removing the token in another tab (a dispatched `storage` event) logs this tab out
- **Commit:** `feat(ui): add auth api and session provider`

### 1.6 UI: protected routes and header

- **Repo:** UI · **PLAN:** §3 (Logout, Expired session), §6 (Auth)
- **Changes:**
  - `ProtectedRoute`: shows a loading state while `status` is `loading`, and `errors.generic` with a "Try again" button while it's `error`; redirects to `/login` when `anonymous`, keeping the requested location in navigation state so login can return there.
  - A `GuestRoute` (or the same component with a flag) for `/login` and `/signup` that redirects an authenticated user to `/`.
  - `AppLayout` header:
    - anonymous: the "Log in" and "Sign up" links
    - authenticated: the user's name, the dashboard and "New task" links, and a "Log out" button calling `logout()`
  - i18n keys for the header go in `common.json`.
- **Acceptance:** tests show:
  - a protected route redirects to `/login` without a token
  - `/login` redirects to `/` with a valid session
  - the header shows the user's name and "Log out" when logged in, and clicking it ends on `/login` with the token gone
- **Commit:** `feat(ui): add protected routes and session-aware header`

### 1.7 UI: login and signup pages

- **Repo:** UI · **PLAN:** §3, §6 (Auth)
- **Changes:**
  - `src/i18n/locales/{en,pt-BR}/auth.json`: register the `auth` namespace in `src/i18n/index.ts`, `i18next.d.ts`, and `locales.test.ts`. Move `pages.login` and `pages.signup` out of `common.json`. Later waves register their namespaces the same way.
  - `LoginPage`: email and password, submit with a mutation, disable the button while pending. On success, `login()` and navigate to the location saved by `ProtectedRoute`, or `/`. With `?expired=1`, show the "session expired" banner (`errors.UNAUTHORIZED`). A `BAD_CREDENTIALS` error shows a form-level message. Keep the "Forgot your password?" link.
  - Both pages trim the email before sending (D4).
  - `SignupPage`: email, name, and password. Client-side checks mirror D5, counting password bytes with `TextEncoder`. On success, `login()` and navigate to `/`. `EMAIL_ALREADY_USED` shows on the email field. `VALIDATION_ERROR` maps each `ApiError.errors` entry to the localized `auth.fieldErrors.<field>` message on that field, never the server's English text.
  - Links between the two pages. Labels, placeholders, and messages all come from `auth.json` or `errors.json`.
- **Acceptance:** component tests:
  - login success stores the token and navigates to `/`
  - login after a redirect returns to the originally requested page
  - a 401 `BAD_CREDENTIALS` shows the localized message, in both languages
  - signup shows the 409 error on the email field
  - server field errors land on the matching fields as localized messages
  - a password of 72 characters but more than 72 bytes is rejected client-side
  - `/login?expired=1` shows the expired-session banner
  - `auth.json` has the same keys in both locales
- **Commit:** `feat(ui): add login and signup pages`

> **Checkpoint B (wave done):** run `docker compose up --build`. Sign up, log out, log in, reload (the session survives), and switch to PT-BR. Replace the token in `localStorage` with garbage and navigate: you land on `/login` with the expired banner. Run `./mvnw test` in the API and `npm run lint && npm run build && npm test` in the UI. Then commit the submodule pointers in the root: `chore: bump submodules (wave 1)`.

## After this wave

- Add the D5 field limits and the D4 case-insensitive email rule to PLAN §3, and the D3 timestamp rule to `AGENTS.md` (API conventions).
- Mark this plan as done.
