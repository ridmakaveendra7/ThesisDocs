# FinSec Phase 1 (base app) — specification

Phase 1 is the vulnerability-carrying seed handed to students to build the
real FinSec app on top of. This is the **full, detailed, from-scratch build
spec** — every feature, every vulnerability it carries, and the order to
build them in. It reflects Phase 1's current, correct scope: a real
frontend (React + Vite) and a deliberately-chosen database (PostgreSQL), not
just a bare JSON API — see
[`docs/Weekly Progress/week-04.md`](docs/Weekly%20Progress/week-04.md) for
that reasoning — and its exact vulnerability set matches
[`finsec_feature_vuln_map.md`](finsec_feature_vuln_map.md)'s Phase 1 tables,
which are the source of truth for *which* vulnerabilities exist; this
document is the source of truth for *how* and *in what order* to build
them. (`working_spec_finsec.md` predates this document and is now stale for
Phase 1 — it still describes the old backend-only, no-frontend design.)

Every vulnerability below is tagged with its row from
`finsec_feature_vuln_map.md`'s Phase 1 section (Backend / Frontend /
Database) so the two documents stay traceable to each other.

## Implementation convention: numbered vulnerability comments

**Every line (or block) of code that carries a vulnerability gets a
comment marking it, numbered sequentially in the order the vulnerabilities
appear in this document (step 0 → step 6).** Format:

```java
// Vuln 1 - Permissive CORS configuration
configuration.setAllowedOriginPatterns(List.of("*"));
```

```properties
# Vuln 2 - Verbose error responses
server.error.include-stacktrace=always
```

Use the comment syntax of whatever file it's in (`//` in Java/JS,
`#`/`{/* */}` in properties/JSX where appropriate). If one vulnerability
spans more than one line or more than one file (e.g. the Actuator finding
touches both a properties file and a security-config rule), tag every
touchpoint with the **same** number — the number identifies the
vulnerability, not the line. Each per-step section below states the exact
number and name to use.

**Master list** (15 numbered vulnerabilities). Three of these (13–15) don't
have a real line of *vulnerable* code to tag — one is an absence, two are
already-true infrastructure states — so for those, add a deliberate,
commented-out line that makes the finding visible in the codebase anyway
(either the vulnerable line if the guardrail were ever flipped on, or an
explanatory annotation on the relevant existing line), carrying the same
numbered comment:

| # | Name | CWE | Step |
|---|---|---|---|
| 1 | Permissive CORS configuration | CWE-942 | 0 |
| 2 | Verbose error responses | CWE-209/756 | 1 |
| 3 | CSRF protection disabled | CWE-352 | 1 |
| 4 | Exposed, unauthenticated Actuator | CWE-200/16 | 1 |
| 5 | Hardcoded/committed database credential fallback | CWE-798 | 2 |
| 6 | Reflected XSS in registration confirmation | CWE-79 | 3 |
| 7 | DOM-based XSS via `dangerouslySetInnerHTML` | CWE-79 | 3 |
| 8 | Hardcoded JWT signing secret | CWE-798 | 4 |
| 9 | SQL injection in the login lookup | CWE-89 | 4 |
| 10 | Log injection in login audit logging | CWE-117-adjacent | 4 |
| 11 | JWT stored in `localStorage` | CWE-522 | 4 |
| 12 | SSRF via webhook URL verification | CWE-918 | 6 |
| 13 | Missing Content-Security-Policy | OWASP A05-adjacent | 3 |
| 14 | Database container network exposure (guardrail) | CWE-284-adjacent | 2 |
| 15 | Database user privileges (superuser, not least-privilege) | CWE-250-adjacent | 2 |

**Still not numbered:** the Jazzer `@FuzzTest` (step 5) isn't a
vulnerability itself — it's a test verifying robustness against malformed
input at the JWT-validation path. Don't tag it with a vuln number; a plain,
non-numbered comment explaining what it's testing is still good practice.

## 1. Starting point — what already exists

Before any of the work below: `finapp/` has a working Spring Boot skeleton
with `POST /api/auth/register` and `POST /api/auth/login`, backed by
PostgreSQL via `docker-compose` (already running, already connected) — but
**none of it is vulnerable yet**. Specifically, as of today:

- Registration returns a plain JSON/text success message (no HTML, no
  reflected username).
- The JWT signing secret is read from an environment variable
  (`${JWT_SECRET}`) with no hardcoded fallback anywhere in tracked source.
- There's no logging of login attempts at all.
- There's no CORS configuration, no Actuator dependency, and CSRF is
  already disabled (this one pre-exists and is correct as-is — see step 1).
- There's no frontend of any kind — `finapp/frontend/` doesn't exist.
- The database password is only ever supplied via `${DB_PASSWORD}` from the
  gitignored `.env` — no committed fallback.
- No `/api/users/search`, `/api/reports/download`, `/api/notifications/webhook-url`,
  or `/actuator/**` endpoints exist.

Everything below is new work, built in the order given — **not** a
description of what to change in existing vulnerable code, because there
isn't any yet.

## 2. Feature-and-vulnerability overview

| Step | Feature | Layer(s) it touches | Vulnerabilities carried |
|---|---|---|---|
| 0 | Bootstrap: frontend scaffold + connectivity check | Backend, Frontend | Permissive CORS (CWE-942) |
| 1 | Application security baseline | Backend | Verbose errors (CWE-209/756), CSRF disabled (CWE-352, pre-existing), Exposed Actuator (CWE-200/16) |
| 2 | Database baseline | Database | Committed DB credential fallback (CWE-798); network-exposure and privilege-scope items are guardrail checks, not new code |
| 3 | User registration | Backend, Frontend | Reflected XSS in confirmation (CWE-79, backend); DOM-based XSS via `dangerouslySetInnerHTML` (CWE-79, frontend); missing CSP (frontend) |
| 4 | User login | Backend, Frontend, Database | Hardcoded JWT secret (CWE-798); SQL injection in the username lookup (CWE-89); log injection in login audit logging (CWE-117-adjacent); JWT kept in `localStorage` (CWE-522, frontend) |
| 5 | Authenticated session validation | Backend | Jazzer fuzz test against JWT validation (no new vuln — a testing requirement) |
| 6 | Notification webhook registration | Backend, Frontend | SSRF (CWE-918) |

## 3. Implementation order — and why it's this order

Build in this exact order; each step either has to exist before the next one
is testable, or is the natural place to make a decision that later steps
depend on.

1. **Bootstrap comes first, before any feature**, because there is
   currently no frontend at all, and every later step needs to be verified
   through the browser, not just `curl`. A frontend calling a backend on a
   different port is a **cross-origin request**, which the browser blocks
   by default — so bootstrapping also means wiring *some* CORS policy right
   away, and the one Phase 1 wires is the deliberately permissive one.
   Without this step, nothing else can be demonstrated end-to-end.
2. **Application security baseline second**, because CSRF, verbose errors,
   and the exposed Actuator are all one-time, app-wide `SecurityConfig`/
   properties changes with no dependency on any feature existing yet —
   cheapest to do in one pass before feature endpoints exist to test
   against them.
3. **Database baseline third**, for the same reason — it's config, not a
   feature, and doing it now means every later step already runs against
   the final database setup rather than a temporary one.
4. **Registration fourth**, because login (step 5) needs a user to exist
   first.
5. **Login fifth**, because the session-validation step and the webhook
   step both need a working JWT to test against.
6. **Session validation sixth** — this is what every authenticated request
   from here on already goes through (nothing new to wire beyond the fuzz
   test), so it's confirmed right after login produces its first real
   token.
7. **Webhook registration last**, because it's the only remaining feature
   and it requires a logged-in session (an `Authorization` header) to
   reach.

## 4. Step 0 — Bootstrap: frontend scaffold + connectivity check

Goal: prove the frontend and backend can talk to each other at all, before
building anything that depends on that working.

### Backend + DB

- Add a trivial, unauthenticated `GET /api/hello` endpoint (a plain
  `ResponseEntity.ok("FinSec backend is up")` or similar, ~3 lines) —
  **not itself a vulnerability carrier**, purely a connectivity-check
  target. `/api/auth/register` and `/api/auth/login` aren't good candidates
  for this because they need a request body and real side effects; this
  needs to be a trivial `GET`.
- Add a `CorsConfigurationSource` bean and wire it into the
  `SecurityFilterChain` via `.cors(...)`:
  ```java
  CorsConfiguration configuration = new CorsConfiguration();
  configuration.setAllowedOriginPatterns(List.of("*"));
  configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
  configuration.setAllowedHeaders(List.of("*"));
  configuration.setAllowCredentials(true);
  ```
  (`setAllowedOriginPatterns`, not `setAllowedOrigins` — Spring rejects a
  literal `"*"` origin combined with credentials at startup, but accepts
  the pattern form, which is equally permissive and the realistic way this
  misconfiguration actually ships.)
  **This is `finsec_feature_vuln_map.md`'s "Cross-origin request (CORS)
  settings" row** (CWE-942) — any website can call the API using a logged-in
  user's credentials. It has to exist this early because without *some*
  CORS policy, step 0's own connectivity check would fail in the browser —
  so the vulnerable version is what gets built, not a placeholder later
  swapped out. → tag the `CorsConfiguration` block with
  `// Vuln 1 - Permissive CORS configuration`.

### Frontend

- Scaffold a Vite + React app (plain JavaScript) at `finapp/frontend/`.
- A single page that calls `GET http://localhost:8080/api/hello` on load
  and renders the response text on screen.
- **Verification for this step:** running `npm run dev` (frontend) and
  `mvn spring-boot:run` (backend) separately, opening the frontend in a
  browser, and seeing the backend's response appear — with the browser's
  network tab confirming the response carries permissive
  `Access-Control-Allow-Origin`/`Access-Control-Allow-Credentials` headers.

## 5. Step 1 — Application security baseline

Goal: set the app-wide security posture Phase 1 intends to ship with —
entirely backend config, no feature code.

### Backend + DB

- **Verbose errors** (`finsec_feature_vuln_map.md`: "How errors are shown
  to the user", CWE-209/756) — in `application.properties`:
  ```properties
  server.error.include-stacktrace=always
  server.error.include-message=always
  server.error.include-binding-errors=always
  ```
  Nothing to trigger this against yet — step 3 (registration) adds the
  first endpoint that throws an exception worth seeing verbose output from.
  → tag each of the three properties lines with
  `# Vuln 2 - Verbose error responses`.
- **CSRF disabled** ("Cross-site request forgery (CSRF) protection",
  CWE-352) — already true in the existing `SecurityConfig`
  (`csrf(csrf -> csrf.disable())`). Nothing to change except adding the
  comment; document it as an existing, intentional finding. Its real-world
  exploitability depends on step 4's decision to use header-based (not
  cookie-based) auth — confirmed there, not here. → tag that line with
  `// Vuln 3 - CSRF protection disabled`.
- **Exposed Actuator** ("Built-in app monitoring/debug endpoints",
  CWE-200/16):
  - Add the `spring-boot-starter-actuator` dependency.
  - `management.endpoints.web.exposure.include=*` in `application.properties`
    — tag with `# Vuln 4 - Exposed, unauthenticated Actuator`.
  - In `SecurityConfig`, add `.requestMatchers("/actuator/**").permitAll()`
    to the authorization rules — a deliberate choice, not an accident of a
    missing rule. Tag this line with the **same** number,
    `// Vuln 4 - Exposed, unauthenticated Actuator` — one vulnerability,
    two touchpoints.

### Frontend

No dedicated page. This step is verified directly against the backend
(`curl localhost:8080/actuator/env`, a malformed request to see the stack
trace) — there's nothing here for the frontend to render or exercise yet.

## 6. Step 2 — Database baseline

Goal: the database-layer findings from `finsec_feature_vuln_map.md`'s Phase
1 "Database" table, beyond the login-lookup SQL injection (which belongs
with step 4, since it's part of the login feature itself).

### Backend + DB

- **Database credentials configuration** (CWE-798) — the only one of the
  three DB rows that's actually new code/config. Today,
  `application-docker.properties` requires `${DB_PASSWORD}` with no
  fallback (nothing for Semgrep/SonarQube to see — the value only ever
  lives in the gitignored `.env`). Change it to a fallback that's visible
  in the tracked properties file:
  ```properties
  spring.datasource.password=${DB_PASSWORD:postgres}
  ```
  Same class of finding as the JWT-secret carrier (step 4) — a literal
  default value in a scanned file, not just an unresolvable env-var
  reference. → tag that line with
  `# Vuln 5 - Hardcoded/committed database credential fallback`.
- **Database container network exposure** (Vuln 14) — a guardrail, not a
  live finding: `docker-compose.yml` already keeps Postgres on the internal
  Docker network only. Make that deliberate and visible by adding the
  vulnerable line **commented out**, right in the `db` service, instead of
  leaving the guardrail only documented in prose:
  ```yaml
  db:
    image: postgres:17
    container_name: finapp-postgres-docker
    # Vuln 14 - Database container network exposure if this port mapping
    # were ever enabled: it would make Postgres directly reachable from
    # outside Docker's internal network, bypassing the app entirely.
    # ports:
    #   - "5432:5432"
    environment:
      ...
  ```
  The only "work" here is adding that comment block and confirming the
  `ports:` mapping stays commented out — the moment it's uncommented, this
  stops being a guardrail and becomes a live finding.
- **Database user privileges** (Vuln 15) — already true today, so there's
  no alternate line to comment out; instead, annotate the existing line
  that causes it:
  ```yaml
  environment:
    POSTGRES_DB: ${POSTGRES_DB}
    # Vuln 15 - Database user privileges: the app connects as this same
    # POSTGRES_USER, which Postgres treats as the database superuser here
    # — full rights over the whole database rather than a role scoped to
    # just the tables finapp needs.
    POSTGRES_USER: ${POSTGRES_USER}
    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
  ```

### Frontend

No dedicated page — none of these three rows have a frontend-visible
surface.

## 7. Step 3 — User registration

Goal: the registration feature, carrying both halves of the
registration/XSS chain — the backend's reflected XSS and the frontend's
DOM-based XSS that renders it.

### Backend + DB

- `POST /api/auth/register`, `produces = MediaType.TEXT_HTML_VALUE`. Keep
  the existing user-creation logic (hash the password, save the `User`,
  throw a plain `RuntimeException` on a duplicate username — this is what
  step 1's verbose-error config now surfaces to the client for the first
  time).
- Change the response body from a static success string to an HTML
  confirmation that includes the submitted username **unescaped**:
  ```java
  String confirmation = "<h1>Welcome, " + registerRequest.getUsername() + "!</h1>"
      + "<p>Your FinSec account has been created.</p>";
  return ResponseEntity.ok(confirmation);
  ```
  **This is `finsec_feature_vuln_map.md`'s "User registration and login"
  row's XSS half** (CWE-79) — a JSON response wouldn't be exploitable in a
  browser; the `text/html` content type is what makes this a real carrier.
  → tag the `confirmation` string-building line with
  `// Vuln 6 - Reflected XSS in registration confirmation`.

### Frontend

- A registration form (username, password) that `POST`s to
  `/api/auth/register` and takes the raw HTML string back.
- Renders that response using `dangerouslySetInnerHTML` instead of treating
  it as plain text:
  ```jsx
  <div dangerouslySetInnerHTML={{ __html: confirmationHtml }} />
  ```
  **This is `finsec_feature_vuln_map.md`'s "Frontend rendering of the
  registration confirmation" row** (CWE-79, DOM-based) — the same backend
  bug, now demonstrable end-to-end through a real browser, not just `curl`.
  → tag the `dangerouslySetInnerHTML` line with
  `// Vuln 7 - DOM-based XSS via dangerouslySetInnerHTML`.
- No Content-Security-Policy is added anywhere (no meta tag, no response
  header). **This is the "Browser security headers (Content-Security-Policy)"
  row** (Vuln 13) — its relevance is demonstrated *here*: nothing stops the
  script injected via the above from running, because there's no CSP
  restricting it. Make the absence deliberate and visible in
  `finapp/frontend/index.html`'s `<head>`, rather than just an unstated gap:
  ```html
  <!-- Vuln 13 - Missing Content-Security-Policy (intentionally left unset,
       so the DOM-XSS payload from Vuln 7 runs with no restriction). A real
       CSP would look like: -->
  <!-- <meta http-equiv="Content-Security-Policy" content="default-src 'self'"> -->
  ```

## 8. Step 4 — User login

Goal: the login feature, carrying four separate findings — this is the
densest step.

### Backend + DB

- **Hardcoded JWT secret** (CWE-798) — replace the current
  `@Value("${jwt.secret}")`-based key with a literal constant in
  `SecurityConfig`:
  ```java
  private static final String JWT_SECRET =
      "Q7mK9xV2pL8sR4nT6wY3cF1hJ5zB0dG8uA2eM7qX9kP4vN6sW1rC3tH8yZ5fL0";
  ```
  used directly to build the `SecretKey` bean. Not read from configuration
  at all — Phase 1 has no deployment story that needs externalizing it, so
  there's no reason to keep the indirection only to leave it unused. → tag
  the `JWT_SECRET` constant with `// Vuln 8 - Hardcoded JWT signing secret`.
- **SQL injection in the login lookup** (CWE-89) — the part of login that
  looks up a user by username (today: `UserService.loadUserByUsername`,
  via the safe `UserRepository.findByUserName`) gets replaced with a
  hand-built native query:
  ```java
  String sql = "SELECT * FROM users WHERE user_name = '" + username + "'";
  ```
  run via `EntityManager.createNativeQuery(sql, User.class)` instead of the
  parameterized repository method. As noted when this was decided: this
  doesn't bypass the password check itself (BCrypt comparison still happens
  in Java code against the hash, not in SQL) — but it's still a genuine
  SQL-injection surface (UNION-based data extraction, error-based schema
  leakage) at a feature every phase needs, which is why it replaced the
  earlier "search users by name" design. → tag the `sql` string-building
  line with `// Vuln 9 - SQL injection in the login lookup`.
- **Log injection in login audit logging** — there is currently no login
  logging at all; add it, unescaped:
  ```java
  log.info("Login attempt for user: " + request.getUsername());
  ```
  A username containing embedded newlines (`\n`/`\r`) can forge additional,
  fake-looking log lines. **This is the "Logging login attempts (audit
  trail)" row.** → tag that line with
  `// Vuln 10 - Log injection in login audit logging`.

### Frontend

- A login form (username, password) that `POST`s to `/api/auth/login` and
  receives `{ "token": "..." }`.
- Stores the token in `localStorage`, not a cookie, and every later
  authenticated frontend call reads it from there to set
  `Authorization: Bearer <token>`. **This is the "Where the login session
  token is kept in the browser" row** (CWE-522) — readable by any script on
  the page, which is what makes it a real target for the XSS from step 3.
  → tag the `localStorage.setItem(...)` call with
  `// Vuln 11 - JWT stored in localStorage`.
- **Verification for this step** doubles as confirming a design decision
  from earlier research: because auth never rides on a cookie, step 1's
  CSRF-disabled finding stays non-exploitable in practice even with a real
  frontend in place — a browser can never attach the bearer header on its
  own.

## 9. Step 5 — Authenticated session validation

Goal: the JWT-validation path every protected endpoint already goes
through by virtue of Spring Security's OAuth2 resource-server config — no
new production code, but a required test.

### Backend + DB

- Add `com.code-intelligence:jazzer-junit` as a test-scope Maven dependency.
- A JUnit 5 `@FuzzTest` that builds a `JwtDecoder` using the same hardcoded
  secret from step 4, and calls `.decode(fuzzedString)` directly with
  fuzzer-supplied input — malformed base64 segments, wrong `.`-part counts,
  truncated/oversized tokens. Catch and ignore `JwtException` (the correct,
  expected outcome for malformed input); anything else is a real finding.
  Bound it with `@FuzzTest(maxDuration = "30s")`.
  **This is the "Validating the session token on every request" row.**
- A new `fuzz` stage/job in `.gitlab-ci.yml` with `when: manual` — an
  on-demand job, not run on every push.

### Frontend

No dedicated page — this path is exercised by every authenticated call the
frontend already makes, starting with step 6's webhook registration.

## 10. Step 6 — Notification webhook registration

Goal: the last feature, and the only remaining vulnerability.

### Backend + DB

- `POST /api/notifications/webhook-url`, body `{ "webhookUrl": "..." }`,
  requires authentication (falls under `anyRequest().authenticated()`,
  unlike `/api/auth/**`).
- Immediately makes a server-side HTTP call to the given URL "to verify
  it's reachable," with no allowlist:
  ```java
  restTemplate.getForEntity(request.getWebhookUrl(), String.class);
  ```
  **This is the "Register a webhook URL for account notifications" row**
  (CWE-918, SSRF) — the server becomes a proxy for reaching addresses the
  submitter couldn't reach directly (e.g. `/actuator/env` from step 1, or a
  cloud metadata endpoint). → tag the `restTemplate.getForEntity(...)` call
  with `// Vuln 12 - SSRF via webhook URL verification`.

### Frontend

- A webhook-registration form on the dashboard (only reachable once logged
  in — this is the first feature that actually requires the stored
  `Authorization` header from step 4 to work), submitting to
  `/api/notifications/webhook-url`.

## 11. Verification checklist

- [ ] The frontend loads and displays the response from `GET /api/hello`,
      with the browser network tab showing permissive CORS headers.
- [ ] `/actuator/env` and `/actuator/heapdump` return data without a token.
- [ ] A malformed request (e.g. bad JSON to `/api/auth/login`) returns a
      body containing a Java stack trace / exception message.
- [ ] Registering with the username `<img src=x onerror=alert(1)>` through
      the **frontend** registration page (not curl) executes script in the
      browser.
- [ ] Registering the same username twice returns a stack trace (step 1's
      verbose-error config, triggered here).
- [ ] The JWT works correctly with no externally configured secret,
      proving the hardcoded value is what's actually signing tokens.
- [ ] Submitting a crafted username at login (e.g. a UNION-based payload)
      demonstrates the SQL injection in the login lookup.
- [ ] A username containing a newline, submitted at login, produces a
      forged-looking extra line in the application log.
- [ ] After logging in through the frontend, the JWT is visible in the
      browser's `localStorage` (DevTools → Application → Local Storage).
- [ ] With the frontend dev server and backend on different origins, a
      cross-origin credentialed `fetch()` against an authenticated endpoint
      succeeds.
- [ ] The JWT-validation `@FuzzTest` runs locally (`JAZZER_FUZZ=1`) and
      terminates on its own within the configured `maxDuration`.
- [ ] The `fuzz` CI job appears as a manual (play-button) job and does not
      run automatically on a normal push.
- [ ] `POST /api/notifications/webhook-url` (while logged in, via the
      frontend) with an attacker-controlled/OOB URL triggers an outbound
      request from the server.
- [ ] `docker-compose.yml`'s `db` service still has no `ports:` mapping to
      the host (the network-exposure guardrail hasn't regressed).
- [ ] `mvn test` runs and passes with no Docker/Postgres container running
      (Spring Boot's auto-configured embedded H2 fallback).

## 12. Traceability to `finsec_feature_vuln_map.md`

| Vuln # | Spec section | `finsec_feature_vuln_map.md` Phase 1 row |
|---|---|---|
| 1 | Step 0 — CORS | Backend: "Cross-origin request (CORS) settings" |
| 2 | Step 1 — verbose errors | Backend: "How errors are shown to the user" |
| 3 | Step 1 — CSRF disabled | Backend: "Cross-site request forgery (CSRF) protection" |
| 4 | Step 1 — Actuator | Backend: "Built-in app monitoring/debug endpoints" |
| 5 | Step 2 — DB credentials | Database: "Database credentials configuration" |
| 14 | Step 2 — DB network exposure (commented-out guardrail) | Database: "Database container network exposure (docker-compose)" |
| 15 | Step 2 — DB user privileges (annotation on existing line) | Database: "Database user privileges" |
| 6 | Step 3 — reflected XSS | Backend: "User registration and login" (XSS half) |
| 7 | Step 3 — DOM XSS | Frontend: "Frontend rendering of the registration confirmation (React SPA)" |
| 13 | Step 3 — missing CSP (commented-out fix) | Frontend: "Browser security headers (Content-Security-Policy)" |
| 8 | Step 4 — hardcoded JWT secret | Backend: "User registration and login" (secret half) |
| 9 | Step 4 — SQL injection | Database: "Looking up a user by username during login" |
| 10 | Step 4 — log injection | Backend: "Logging login attempts (audit trail)" |
| 11 | Step 4 — localStorage token | Frontend: "Where the login session token is kept in the browser" |
| — | Step 5 — Jazzer fuzz test (a test, not a vuln) | Backend: "Validating the session token on every request" |
| 12 | Step 6 — SSRF | Backend: "Register a webhook URL for account notifications" |
