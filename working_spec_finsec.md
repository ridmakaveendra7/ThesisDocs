# FinSec Phase 1 (base app) — specification

Phase 1 is the vulnerability-carrying seed handed to students to build the
real FinSec app on top of. This spec describes it from scratch — what it must
do and where each vulnerability lives — without assuming or preserving any
particular existing codebase. Build it however's cheapest; nothing here is
tied to any specific file, class, or prior implementation.

It turns the research in
[`docs/Weekly Progress/week-03.md`](docs/Weekly%20Progress/week-03.md) into a
concrete build target: minimum lines of code, maximum findings for Semgrep,
SonarQube, and OWASP ZAP (§3), plus one working fuzz-testing example (§4).
Backend-only (Java/Spring Boot) — no frontend, per that research's scope
note. Every vulnerability below is tagged with its catalog number (`§N`);
read that entry for full detection rationale/sources. Fuzz-testing items are
tagged against the research doc's "Fuzz-testing research" section instead.

## 1. What Phase 1 is

The smallest Spring Boot app that:
1. Lets a user register (username + password) and log in, receiving a JWT —
   just enough auth to gate later endpoints and to double as a vulnerability
   carrier itself.
2. Exposes a handful of small additional endpoints whose sole purpose is to
   carry the vulnerabilities in §3 as cheaply as possible.
3. Ships one working Jazzer fuzz test (§4) against the JWT validation path —
   the one fuzz target guaranteed to exist this early — as the reference
   example for fuzz tests students add alongside their own later features.
4. Does nothing else. No accounts, transactions, payments, or bank
   integration — those are later phases, built by students on top of this.

Total target surface: **6 endpoints**, none more than ~15 lines of handler
code, plus one fuzz test. There is no requirement to match any particular
prior structure — optimize purely for lines-of-code-per-finding. Every
carrier below is chosen to be a feature Phase 1 would plausibly need anyway,
not an invented test endpoint — see §3.3 and §3.8 for why the earlier
`/api/greeting` and `/api/bank/redirect-check` designs were replaced.

## 2. Application surface

| Endpoint | Purpose | Vulnerability carried |
|---|---|---|
| `POST /api/auth/register` | register user | §3.5 hardcoded JWT secret, §3.3 reflected XSS |
| `POST /api/auth/login` | issue JWT | — |
| `GET /api/users/search?name=` | look up users by (partial) name | §3.6 SQL injection |
| `GET /api/reports/download?file=` | download a report/statement file | §3.7 path traversal |
| `POST /api/notifications/webhook-url` | register a webhook URL for account notifications | §3.8 SSRF |
| `/actuator/**` | ops introspection (new dependency) | §3.1 exposed actuator |

## 3. Vulnerabilities to introduce

### 3.1 Actuator exposed and unauthenticated — §1 (CWE-200/CWE-16)
- Add the Actuator starter dependency.
- Expose all endpoints (`management.endpoints.web.exposure.include=*`).
- Explicitly permit-all `/actuator/**` in the security configuration — a
  deliberate choice, not reliance on a framework default.

### 3.2 Permissive CORS — §2 (CWE-942)
- A CORS configuration allowing any origin
  (`setAllowedOriginPatterns(List.of("*"))`, not `setAllowedOrigins`, so
  Spring accepts it at startup) with credentials allowed.

### 3.3 Reflected XSS via the registration confirmation — §7 (CWE-79)
- `POST /api/auth/register`'s response body switches from a static "User
  registered successfully" string to an HTML confirmation that includes the
  submitted username unescaped (e.g. `"<h1>Welcome, " + username + "!</h1>"`),
  with an HTML content type (`text/html`). A JSON response would not be
  exploitable in a browser — the HTML content type is what makes this a real
  XSS carrier.
- Replaces an earlier design that used a dedicated `/api/greeting?name=`
  smoke-test endpoint purely to host this vulnerability. A standalone
  endpoint whose only job is reflecting a query param isn't something a real
  FinSec app would ever have — a registration confirmation is a feature
  every app already needs, so this carries the same finding without
  inventing a fake one. It also removes an endpoint instead of adding one.

### 3.4 Verbose error responses — §3 (CWE-209/CWE-756)
- Enable full stack-trace/message/binding-error output on errors
  (`server.error.include-stacktrace=always`, `include-message=always`,
  `include-binding-errors=always`).
- Registration must throw an unhandled exception on a duplicate username (a
  plain runtime exception is enough) so there's something to trigger this on.

### 3.5 Hardcoded/committed JWT secret — §4/§12 (CWE-798)
- The JWT signing key is a literal string constant in source — not read from
  an environment variable at all. Phase 1 has no deployment story that needs
  externalized config, so there's no reason to add that indirection only to
  leave it unused; a plain hardcoded secret is both simpler and the intended
  finding (a value that only lives in a gitignored `.env` file is invisible
  to Semgrep/SonarQube, per §4's caveat — it must be in a scanned source or
  properties file to count).

### 3.6 SQL injection — §6 (CWE-89)
- `/api/users/search?name=` runs a hand-built SQL string
  (`"SELECT * FROM users WHERE user_name LIKE '%" + name + "%'"`) via a
  native query, instead of a parameterized one. ~8 lines total including the
  endpoint.

### 3.7 Path traversal — §8 (CWE-22)
- `/api/reports/download?file=` resolves the given filename directly against
  a fixed reports directory (`new File(reportsBaseDir, file)`) with no
  normalization or allowlist, then streams it back.
- Ship one or two placeholder files in that directory so there's something
  legitimate to fetch by name first.

### 3.8 SSRF — §9 (CWE-918)
- `POST /api/notifications/webhook-url`, body `{ "webhookUrl": "..." }`,
  registers a URL the user wants account notifications sent to, and
  immediately makes a server-side HTTP call to it (to "verify it's
  reachable") with no allowlist.
- Replaces an earlier design that used `/api/bank/redirect-check` to stand in
  for validating the spec's `TPP-Redirect-URI`. That framing presupposes
  EuroTrust Bank integration, which doesn't exist until Phase 2 — a Phase 1
  feature shouldn't be justified by a Phase 2 dependency. A notification
  webhook needs no bank at all: it's FinSec's own feature end-to-end, and
  "register a URL, then have the server call it" is exactly the same SSRF
  mechanism (an unvalidated server-side fetch of a user-supplied URL), just
  hung on a carrier Phase 1 can actually justify on its own.

### 3.9 CSRF disabled — §5 (CWE-352)
- CSRF protection is disabled in the security configuration rather than
  configured — one line, and a well-known SonarQube security hotspot
  (commonly S4502).

### 3.10 (Optional, zero-cost) No rate limiting on login
- No throttling on `/api/auth/login`. Free — it's an absence, not an
  addition — but not reliably caught by any of the three tools' default
  configuration (§14's blind-spot note), so it doesn't count toward the
  tool-detection goal. Worth noting for a later manual/threat-modeling
  exercise, not a build requirement.

## 4. Fuzz testing in Phase 1

Separate testing dimension from the Semgrep/SonarQube/ZAP vulnerability set
in §3 — see
[`Weekly Progress/week-03.md`](docs/Weekly%20Progress/week-03.md#fuzz-testing-research)
("Fuzz-testing research") for the full tooling writeup and the complete
candidate list across the *full* completed FinSec system. Most of those
candidates (the §8.1 signature-format parser, payment fields, nonce
handling, the `state` parameter, bank-response parsing) depend on features
Phase 1 deliberately doesn't build, so they stay deferred until whichever
later phase builds the feature they'd fuzz (§4.3). One candidate is
different: JWT/bearer-token validation already exists in Phase 1 — it's
exactly what `/api/auth/login` issues and every non-auth endpoint checks —
so Phase 1 ships it as a working example, not just a described target.

### 4.1 Required: a Jazzer fuzz test for JWT validation
- A JUnit 5 fuzz test (`com.code-intelligence:jazzer-junit` as a test-scope
  Maven dependency) with an `@FuzzTest` method that calls the token-decoding/
  validation logic directly with fuzzer-supplied strings — malformed base64
  segments, wrong `.`-part counts, truncated/oversized tokens, unexpected
  claim types.
- Bounded per-method, not left open-ended: set `maxDuration` (e.g. `"30s"`)
  and/or `maxExecutions` on the `@FuzzTest` annotation so a run has a
  predictable worst-case cost — see the research doc's CI/pipeline note for
  why this matters here specifically.
- This is the one fuzz test Phase 1 must ship. Its main job is being a
  working, copyable example — students add more `@FuzzTest` methods the same
  way as they build their own parsing/validation logic in later phases.

### 4.2 CI wiring
- A new `fuzz` stage/job in `.gitlab-ci.yml` with `when: manual` — an
  on-demand job a student triggers themselves from the pipeline UI, not a
  per-push or scheduled/nightly job. (Ordinary `when: manual` on a normal job
  in the same pipeline is fully supported; the unsupported case is
  `when: manual` on a cross-project *trigger* job, which doesn't apply here.)
- Regression-mode Jazzer (replaying any previously-saved crashing inputs)
  can ride along in the normal `mvn test` step for free — it's cheap and
  bounded by definition. The manual job is specifically for *exploratory*
  fuzzing mode (`JAZZER_FUZZ=1`), which is what actually needs the
  maxDuration/maxExecutions bounds from §4.1 and the on-demand trigger.

### 4.3 Deferred fuzz targets (depend on later-phase features)
Everything else in the research doc's candidate list is out of scope for
Phase 1 specifically because the feature it would fuzz doesn't exist yet:
the §8.1 signature-format parser, payment fields, nonce handling, the
`state` parameter, bank-response parsing, free-text bank data, and the
session channel marker. Add the corresponding fuzz test alongside each
feature as it's built in a later phase, not before — same principle as §5's
deferred vulnerabilities below.

## 5. Deferred / explicitly not in Phase 1 (with reasons)

- **XXE (§10)** — no XML-accepting feature exists; inventing one purely to
  carry this fails the "cheap and plausible" test. Add if/when a bulk-import
  or SEPA-XML feature is ever built.
- **Weak randomness/hash (§11)** — needs a real token/code-generating
  feature (reset code, pairing code) to exist first; Phase 1 generates none.
  Revisit once such a feature is built.
- **Username enumeration (§13)**, **mass assignment**, **IDOR** — logic-only
  items none of the three tools reliably catch by default; not worth the
  extra code in a phase whose entire point is tool-detected findings. Keep as
  candidates for a later manual-testing/threat-modeling exercise.

## 6. Verification checklist

- [ ] `/actuator/env` and `/actuator/heapdump` return data without a token.
- [ ] A cross-origin credentialed `fetch` against an authenticated endpoint
      is allowed by the CORS response headers.
- [ ] Registering with the username `<script>alert(1)</script>` reflects it
      unescaped in the registration confirmation's `text/html` response.
- [ ] Registering the same username twice returns a body containing a Java
      stack trace / exception message.
- [ ] The JWT works correctly with no externally configured secret (proving
      the hardcoded value is what's actually signing tokens).
- [ ] `GET /api/users/search?name=' OR '1'='1` returns all users.
- [ ] `GET /api/reports/download?file=../../../../etc/passwd` (or an
      equivalent app file) returns content outside the reports directory.
- [ ] `POST /api/notifications/webhook-url` with an attacker-controlled/OOB
      URL triggers an outbound request from the server.
- [ ] The JWT-validation `@FuzzTest` runs locally (`JAZZER_FUZZ=1`) and
      terminates on its own within the configured `maxDuration`/
      `maxExecutions` bound rather than running indefinitely.
- [ ] The `fuzz` CI job appears as a manual (play-button) job in the pipeline
      UI and does not run automatically on a normal push.

## 7. Traceability to the research catalog

| Spec item | Catalog entry |
|---|---|
| §3.1 Actuator | `Weekly Progress/week-03.md` §1 |
| §3.2 CORS | §2 |
| §3.3 Reflected XSS | §7 |
| §3.4 Verbose errors | §3 |
| §3.5 Hardcoded JWT secret | §4 / §12 |
| §3.6 SQL injection | §6 |
| §3.7 Path traversal | §8 |
| §3.8 SSRF | §9 |
| §3.9 CSRF disabled | §5 |
| §3.10 No rate limiting | §14 |
| §4.1 JWT fuzz test | `Weekly Progress/week-03.md` "Fuzz-testing research" — JWT/bearer-token validation path (guaranteed auth-area target) |
| §4.2 CI wiring | "Fuzz-testing research" — CI/pipeline note specific to this project |
| §4.3 Deferred fuzz targets | "Fuzz-testing research" — target-mapping table (all other rows) |
