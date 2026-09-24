# Week 5 requirements

## Goal

Finish seeding Phase 1's vulnerability set. Weeks 3–4 did the research and
produced [`phase 1 spec.md`](../../phase%201%20spec.md) — the reviewed,
already-authoritative build spec for all 15 numbered vulnerabilities — and
week 4 implemented Steps 0–1 (Vulns 1, 2, 3, 4, 11; 5 of 15). This week is
**pure implementation against that existing spec**: no new research, no
re-deciding what to build — just building Steps 2–6 exactly as
`phase 1 spec.md` already specifies them, so all 15 rows in
[`finsec_feature_vuln_map.md`](../../finsec_feature_vuln_map.md)'s Phase 1
tables exist in code, not just on paper.

Follow `phase 1 spec.md`'s own step order (2 → 6) and its numbered-comment
convention (`// Vuln N - Name` / `# Vuln N - Name`) for every touchpoint.
Output goes in `docs/Weekly Progress/week-05.md` once done, following the
same shape as `Weekly Progress/week-04.md`'s implementation sections
(what was built, any real problems hit and how they were resolved).

## 1. Step 2 — Database baseline

**What to produce:** Vuln 5 — change
`application-docker.properties`'s `spring.datasource.password` from
`${DB_PASSWORD}` to `${DB_PASSWORD:postgres}` (a committed fallback value,
same carrier class as the JWT secret). Tag with
`# Vuln 5 - Hardcoded/committed database credential fallback`.

Also close the two guardrail rows, still missing from `docker-compose.yml`
as of week 4's implementation:
- **Vuln 14** — add the commented-out `ports: - "5432:5432"` block to the
  `db` service with its explanatory comment, confirming the mapping stays
  disabled.
- **Vuln 15** — annotate the existing `POSTGRES_USER` line noting the app
  connects as the database superuser rather than a least-privilege role.

Neither 14 nor 15 needs new *behavior* — only the deliberate, visible
annotation `phase 1 spec.md` §6 describes.

## 2. Step 3 — User registration

**What to produce:**
- `POST /api/auth/register` changed to `produces = MediaType.TEXT_HTML_VALUE`,
  returning an HTML confirmation with the submitted username concatenated
  in unescaped (Vuln 6, reflected XSS).
- A registration page on the frontend (form → `POST /api/auth/register` →
  render the raw response via `dangerouslySetInnerHTML`, Vuln 7).
- The commented-out CSP annotation in `frontend/index.html`'s `<head>`
  (Vuln 13) — the deliberate-absence marker `phase 1 spec.md` §7 specifies.

This is the first step that needs a real frontend page beyond login, so it's
also the first opportunity to verify the full reflected→DOM XSS chain
through a browser rather than `curl`.

## 3. Step 4 — User login (backend half)

**What to produce:** the three backend findings `phase 1 spec.md` §8
specifies — login's frontend half (Vuln 11, `localStorage`) is already done.
- **Vuln 8** — replace the `@Value("${jwt.secret}")`-sourced key in
  `SecurityConfig` with the literal hardcoded constant.
- **Vuln 9** — replace `UserService.loadUserByUsername`'s use of the safe
  `UserRepository.findByUserName` with the hand-built native query run via
  `EntityManager.createNativeQuery`.
- **Vuln 10** — add unescaped login-attempt audit logging
  (`log.info("Login attempt for user: " + request.getUsername())`).

Verify the SQL-injection and log-injection findings each work end-to-end
(a UNION-based payload at login; a newline-containing username producing a
forged log line) — both are on the verification checklist in
`phase 1 spec.md` §11.

## 4. Step 5 — Authenticated session validation (fuzz test)

**What to produce:**
- Add `com.code-intelligence:jazzer-junit` as a test-scope Maven dependency.
- A JUnit 5 `@FuzzTest(maxDuration = "30s")` building a `JwtDecoder` from
  the same hardcoded secret (Vuln 8) and fuzzing `.decode(...)` directly —
  catching and ignoring `JwtException`, treating anything else as a real
  finding.
- A new `fuzz` stage/job in `.gitlab-ci.yml`, `when: manual` — confirm it
  does not run automatically on a normal push.

Not a new vulnerability — a testing requirement per `phase 1 spec.md` §9.

## 5. Step 6 — Notification webhook registration

**What to produce:**
- `POST /api/notifications/webhook-url` (authenticated, body
  `{ "webhookUrl": "..." }`), immediately calling
  `restTemplate.getForEntity(request.getWebhookUrl(), String.class)` with no
  allowlist (Vuln 12, SSRF).
- A webhook-registration form on the dashboard, reachable only once logged
  in, submitting to that endpoint.

This is the last remaining feature and the only step that depends on a
working authenticated session (Step 4/5) to reach.

## Verification

Work through the remaining, still-unchecked items of `phase 1 spec.md` §11
(the checklist items covering Steps 2–6) — don't re-verify Steps 0–1's items,
already confirmed in week 4.

## Non-goals

- No new vulnerability research or re-scoping of `phase 1 spec.md` — it's
  already reviewed; this week executes it.
- No Phase 2 feature work.
- No OWASP ZAP CI wiring — still open from week 3/4, deferred again; get the
  full vuln set into code first so ZAP (once wired) has a complete Phase 1
  app to scan against.
- No fixing/remediating anything just built — these are intentional
  findings for later tooling to detect, not bugs to resolve.

## Open questions to resolve during implementation

- Does adding `jazzer-junit` (Step 5) introduce any Maven dependency
  conflicts with the existing `spring-boot-starter-test`/Spring Security
  test stack? Confirm before assuming a clean add.
- Does the native-query change in Step 4 (Vuln 9) require any adjustment to
  `UserService`'s Spring Security integration (`UserDetailsService`), or is
  it a drop-in replacement for the existing repository call?
