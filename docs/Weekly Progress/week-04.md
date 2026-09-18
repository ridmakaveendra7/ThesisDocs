# Week 4 progress — frontend decision

**Author:** Ridma Kaveendra

Research output for requirement 1 of
[`Weekly Requirements/week-04.md`](../Weekly%20Requirements/week-04.md) only
— the frontend decision. Requirements 2–4 (database decision, the
frontend/database vulnerability mapping, and the final concrete Phase 1
spec) are not addressed here; they'll get their own pass once this decision
is reviewed.

## Decision

**React 18 + Vite, plain JavaScript (no TypeScript).** A minimal
single-page app, no component library, no state-management library, no HTTP
client dependency (`fetch`, not axios). Would live at `finapp/frontend/`, in
the same git repository as the backend.

## Comparison

| Candidate | Verdict | Reasoning |
|---|---|---|
| **React + Vite** | **Chosen** | Escape-by-default JSX rendering means a DOM-XSS finding requires a deliberate opt-out (`dangerouslySetInnerHTML`) rather than being the structurally forced outcome of the templating model — matching the "every vulnerability is an intentional choice, not an accident" discipline used for every backend finding since week 3. It's also the stack with the most mature security-tooling coverage of the candidates evaluated: Semgrep ships a **named, default-enabled registry rule** for exactly this pattern (`typescript.react.security.audit.react-dangerouslysetinnerhtml`, tagged CWE-79 / OWASP A03:2021 Injection), and `eslint-plugin-react` ships the equivalent `no-danger` lint rule out of the box. Most job-relevant choice for a "here's a realistic app" teaching template. |
| Vue + Vite | Rejected | Same auto-escape-by-default / deliberate-opt-out shape (`v-html`) as React, so no safety-story disadvantage on paper — but Semgrep's Vue/`.vue`-file support has a documented history of being thin: a long-standing GitHub feature request (semgrep/semgrep#1751, opened 2020) tracked Semgrep simply *ignoring* `.vue` files, and single-file-component parsing isn't part of the well-established, named-rule coverage React gets. That's a real gap for a project whose whole thesis is "does the default tool configuration actually catch this." |
| Angular | Rejected | TypeScript-first, with its own DI/module/build ceremony — that boilerplate would dominate a ~5-page app, against the "minimum base code" constraint that's shaped every Phase 1 decision since week 3. |
| Svelte/SvelteKit | Rejected | Genuinely lean, but its compiled output doesn't have the same breadth of default SAST rule coverage that React's JSX gets — no equivalent named registry rule turned up for Svelte's templating. Would leave the frontend under-covered by the same static-analysis story the backend relies on. |
| Vanilla JS/HTML, no framework | Rejected | Lowest possible LOC, but *unescaped-by-default* templating (raw `innerHTML`/string concatenation) makes DOM XSS the obvious, structurally-forced outcome rather than a believable, deliberate mistake — it undersells the "someone had to actively opt out of a safe default" teaching point React/Vue give for free. Also not representative of how a real fintech SPA gets built. |
| Server-rendered (Thymeleaf, inside Spring Boot) | Rejected | No separate frontend at all contradicts the point of choosing one, and doesn't match this project's own architecture: `finsec_app_final_spec.md` §2 models the system as **two separate clients** (app + web) calling one JSON backend across trust boundaries A and B — a real client-side SPA is the accurate shape for the Boundary-B (Web↔Backend) channel; a server-rendered page isn't. |

## Consequences for the open questions week 3 deferred

- **Token transport: bearer JWT in the `Authorization` header, read from
  `localStorage` on each request — not a cookie.** This is a deliberate,
  realistic-but-bad choice (the token is readable by any script on the
  page), picked specifically so it can chain with a DOM-XSS finding into a
  "steal the token via XSS → replay it" narrative later. It also resolves a
  caveat week 3's catalog left open on the CSRF-disabled finding
  (`Weekly Progress/week-03.md` §5): because auth never rides on a cookie, a
  browser can never attach it on its own, so CSRF-disabled stays a
  static/hotspot-only finding even with a real frontend in place — not a
  live CSRF PoC.
- **CORS's live PoC becomes possible, not just a static one.** A frontend
  dev server and the backend naturally run on different origins
  (`http://localhost:5173` vs. `http://localhost:8080`), so the already-seeded
  wildcard-origin-with-credentials CORS misconfiguration (week-3 catalog §2)
  now has a real cross-origin credentialed `fetch()` to demonstrate it from,
  resolving that entry's "no live PoC until a frontend exists" caveat.
- **CSP is not addressed by this decision alone** — nothing about choosing
  React adds or removes a Content-Security-Policy; whether one gets added
  (and if not, whether that's logged as a deliberate "missing security
  headers" finding per week 3's catalog) is still open and belongs with the
  vulnerability-mapping requirement (§3 of the requirements doc), not this
  frontend-stack decision.

## Sources

- [Semgrep Registry — react ruleset](https://registry.semgrep.dev/ruleset/react)
- [react-dangerouslysetinnerhtml — Semgrep rule detail](https://semgrep.dev/r/typescript.react.security.audit.react-dangerouslysetinnerhtml.react-dangerouslysetinnerhtml)
- [semgrep-rules/typescript/react/security/audit/react-dangerouslysetinnerhtml.yaml — GitHub](https://github.com/semgrep/semgrep-rules/blob/develop/typescript/react/security/audit/react-dangerouslysetinnerhtml.yaml)
- [eslint-plugin-react — no-danger rule docs](https://github.com/jsx-eslint/eslint-plugin-react/blob/master/docs/rules/no-danger.md)
- [Vue.js support — semgrep/semgrep issue #1751](https://github.com/semgrep/semgrep/issues/1751)

---

# Week 4 progress — database decision

Research output for requirement 2 of
[`Weekly Requirements/week-04.md`](../Weekly%20Requirements/week-04.md) only
— the database decision, re-derived without treating `finapp/`'s existing
Postgres/`docker-compose` setup as a given.

## Decision

**PostgreSQL 17, containerized via `docker-compose`.**

## Comparison

| Candidate | Verdict | Reasoning |
|---|---|---|
| **PostgreSQL** | **Chosen** | Relational + ACID fits the banking domain the app is heading toward: Phase 2's accounts → transactions → payments model needs foreign-key integrity and transactional writes for money movement, and the derived metrics (`finsec_app_final_spec.md` §5.4) are aggregate SQL queries across that structure. A real, separately-containerized DB (vs. embedded) also carries the infra-security concerns this thesis's CI/CD focus cares about — externalized credentials, a network-reachable DB container. Mature Spring Data JPA support. On direct security-tooling comparison with MySQL, both have an official **CIS Benchmark** (CIS PostgreSQL Benchmark and CIS Oracle MySQL Benchmark both exist and are actively maintained), so that alone isn't a differentiator — but PostgreSQL has **native row-level security (RLS)**, letting per-row access policies be enforced at the database layer; MySQL has no equivalent feature and can only approximate it with views/`SQL SECURITY DEFINER` tricks. That's a genuine forward-looking edge for Phase 2's access-matrix requirement (§5.3 of the final spec — enforcing what a web-channel vs. app-channel session may read/write), where a defense-in-depth DB-level policy is a real option on Postgres and not a first-class one on MySQL. |
| MySQL/MariaDB | Rejected (close second) | Functionally near-equivalent for Phase 1's needs and equally well covered by a CIS hardening benchmark — not rejected for lacking security tooling. Rejected on native feature fit: no built-in row-level security (see above), and weaker native JSON/array typing than Postgres's `jsonb`/array types, which are a better fit if the bank's `tppMessages` error shape or similar structured data ever needs to be persisted rather than just proxied. |
| H2 (embedded) | Rejected as primary, kept for tests | Zero-setup and matches the original reference material's own base design, but an in-memory DB with no separate container hides exactly the infra-level concerns (credential handling, network exposure) this project wants the exercise to carry — it doesn't reflect a real deployment. Confirmed as a genuinely free addition for testing, though: Spring Boot auto-configures an embedded database automatically whenever an embedded driver (H2/HSQL/Derby) is on the classpath and no `DataSource` bean or `spring.datasource.*` properties are defined — no custom test-profile wiring needed, `mvn test` would just work without Docker as long as H2 is a test-scope dependency. |
| SQLite | Rejected | Too lightweight for a "server with its own container to misconfigure" story central to this project's infra-security angle; weaker concurrent-write story once Phase 2 adds real write traffic (payments). |
| MongoDB (or other NoSQL) | Rejected | Not rejected for lacking transactional integrity — MongoDB has supported multi-document ACID transactions since v4.0 (replica sets) and v4.2 (sharded clusters), so that argument doesn't hold up. Rejected instead on structural domain fit: this app's core relationships (users → accounts → transactions → payments) and its aggregate derived-metrics queries are naturally expressed as relational joins and `GROUP BY`-style aggregation: forcing that into a document model would mean either denormalizing (duplicating account/user data across transaction documents) or replicating join logic in application code — added complexity with no corresponding benefit here, since nothing in the spec calls for the schema flexibility a document store is actually good for. |

## Consequences / open items this decision leaves for later requirements

- **H2 as a test-only dependency is a genuinely free addition**, not a
  design compromise — it changes nothing about the primary Postgres
  decision above, and doesn't need its own profile file given Spring Boot's
  auto-configuration behavior. Whether to actually add it is implementation
  work, not part of this decision.
- **Native PostgreSQL row-level security** is noted here as a real option
  for Phase 2's access-matrix enforcement, but whether to actually use it
  (vs. enforcing entirely in the Spring Security/service layer, as
  `finsec_app_final_spec.md` §5.3 currently implies with "all these checks
  happen server-side") is a Phase 2 design decision, not something this
  week's database choice commits to.
- **Whether any DB-related items belong in the Phase 1 build** (e.g. a
  seeded credential-handling finding, as raised in the requirements doc's
  §3 candidate list) is explicitly out of scope for this section — that's
  requirement 3's vulnerability-mapping work, not this one.

## Sources

- [CIS PostgreSQL Benchmarks — cisecurity.org](https://www.cisecurity.org/benchmark/postgresql)
- [CIS Oracle MySQL Benchmarks — cisecurity.org](https://www.cisecurity.org/benchmark/oracle_mysql)
- [Row-security — PostgreSQL wiki](https://wiki.postgresql.org/wiki/Row-security)
- [Row-Level Security in MySQL: How to Build PostgreSQL-like RLS with Views, Functions, and Laravel](https://new2026.medium.com/row-level-security-in-mysql-how-to-build-postgresql-like-rls-with-views-functions-and-laravel-8a7c94b95e77)
- [SQL Databases — Spring Boot reference docs (embedded database auto-configuration)](https://docs.spring.io/spring-boot/reference/data/sql.html)
- [MongoDB's ACID Transaction Guarantee — MongoDB](https://www.mongodb.com/products/capabilities/transactions)
- [Multi-Document Transaction in MongoDB — GeeksforGeeks](https://www.geeksforgeeks.org/mongodb/multi-document-transaction-in-mongodb/)

---

# Week 4 progress — frontend/database vulnerability mapping

Output for requirement 3 of
[`Weekly Requirements/week-04.md`](../Weekly%20Requirements/week-04.md) —
the new vulnerabilities the frontend and database decisions (above) bring,
split by Phase 1 vs. Phase 2.

Added directly to
[`finsec_feature_vuln_map.md`](../../finsec_feature_vuln_map.md) as new rows
in its existing Phase 1 / Phase 2 tables, rather than duplicated here — that
document is the project's single feature-vs-vulnerability map (backend rows
already lived there since before this week), so the frontend/database rows
belong alongside them, not in a second, separate list. Six new rows under
Phase 1's "Features with no counterpart in the reference spec" (the
registration-page DOM-XSS chain, `localStorage` token exposure, missing CSP,
a committed-DB-credential candidate, DB network exposure, and DB user
privilege scope) and eight new rows under Phase 2's equivalent table
(client-side access-control false confidence, frontend SCA-redirect
handling, clickjacking on payment approval, unvalidated cross-window
messaging, client-side caching of financial data, repeated SQL-injection
risk in student-written queries, unencrypted derived-metrics storage, and
backup/snapshot exposure).

**Update, made while building requirement 4's spec below:** two of Phase 1's
rows were reconsidered on relevance grounds and changed after this section
was written — both now reflected directly in `finsec_feature_vuln_map.md`,
not re-described here:
- **"Download a report file by name" moved from Phase 1 to Phase 2.** A
  banking app plausibly has a statement/receipt download feature, but Phase
  1 has no real accounts/transactions for a report to be *about* yet — the
  row now lives in Phase 2's Backend table as "Downloading an account
  statement or payment receipt," where it has real data to reference.
- **"Search for a user by name" (the SQL-injection carrier) was replaced by
  putting the same vulnerability in the login lookup instead.** A
  self-service "search other users by name" feature doesn't hold up as
  something a real banking TPP would ship (see the chat discussion this
  session) — the SQLi finding now lives in `UserService.loadUserByUsername`,
  the query every login already runs, which needs no invented feature to
  justify it.

---

# Week 4 progress — final concrete Phase 1 spec

Output for requirement 4 of
[`Weekly Requirements/week-04.md`](../Weekly%20Requirements/week-04.md) —
turning the frontend decision, database decision, and vulnerability mapping
above into the actual, detailed, from-scratch Phase 1 build spec.

Written to a new top-level document,
[`phase 1 spec.md`](../../phase%201%20spec.md), rather than duplicated here.
`working_spec_finsec.md` — the original Phase 1 spec from before this week —
is now stale (still describes the old backend-only, no-frontend design) and
was deliberately left as-is rather than overwritten; `phase 1 spec.md` is
the current, authoritative one.

## What the spec contains

- **A starting-point section** stating plainly what `finapp/` actually has
  today (a register/login skeleton with *none* of its intended
  vulnerabilities built yet — no hardcoded secret, no HTML/XSS response, no
  audit logging, no CORS, no Actuator, no frontend) — the previous spec
  implied Phase 1 was further along than it is.
- **Seven implementation steps (0–6)**, in dependency order, each split into
  a **Backend + DB** part and a **Frontend** part per the requirement:
  1. Bootstrap — scaffold the React frontend and verify it can reach the
     backend at all, which is also where the permissive CORS bean has to be
     wired (a browser blocks cross-origin calls without *some* CORS policy,
     so this can't be deferred to a later "security config" step).
  2. Application security baseline — verbose errors, CSRF (pre-existing,
     documented not changed), exposed Actuator.
  3. Database baseline — the committed DB-credential fallback (new config);
     network exposure and privilege scope are guardrail/verification items,
     not new code, since both are already true of the current setup.
  4. User registration — reflected XSS (backend) chained into DOM-based XSS
     (frontend, via `dangerouslySetInnerHTML`), plus the missing-CSP finding.
  5. User login — hardcoded JWT secret, the SQL injection (now in the login
     lookup, per the update above), log injection in new login-audit
     logging, and the `localStorage` token-storage decision.
  6. Authenticated session validation — the Jazzer `@FuzzTest` against JWT
     decoding (no new vulnerability, a testing requirement).
  7. Notification webhook registration — SSRF.
- **All 15 rows in `finsec_feature_vuln_map.md`'s current Phase 1 tables are
  covered exactly once**, each tagged inline to the step that builds it and
  cross-referenced in a traceability table at the end.
- **An updated verification checklist** covering every step, including the
  frontend-driven checks (DOM XSS through the real registration page, token
  visible in `localStorage`, a live cross-origin `fetch()` succeeding) and
  the two DB guardrail checks (no published Postgres port, `mvn test`
  passing without Docker via Spring Boot's embedded-H2 fallback).

## Why this order specifically

Bootstrap has to come before every feature because nothing else can be
verified through the browser without it, and it *has* to include the
permissive-CORS wiring rather than deferring it, since the connectivity
check itself would fail cross-origin otherwise. Security and database
baseline configuration come next because they're one-time, feature-independent
settings — cheaper to get out of the way before there's anything to test them
against. Registration precedes login because login needs a user to exist;
login precedes session-validation and webhook registration because both need
a real JWT to test against. Webhook registration is last because it's the
only feature that requires being logged in to reach.

## Non-goals / what's still open

- Implementation hasn't started from this spec yet — it's the plan, not the
  build.
- CI wiring for SonarQube/ZAP and the frontend build stage are still open
  from week 3/`finsec_app_final_spec.md` §5.7 — not addressed by this spec.
- `working_spec_finsec.md` was not reconciled or deleted — it's simply
  superseded for Phase 1 purposes by `phase 1 spec.md`. Worth a decision
  later on whether to retire it or fold it in, but out of scope for this
  week.

---

# Week 4 progress — frontend scaffolding

Output for requirement 5 of
[`Weekly Requirements/week-04.md`](../Weekly%20Requirements/week-04.md)
(added retroactively — see that section's own note on why).

Built at `finapp/frontend/`: React 18 + Vite, pinned to stable versions
(`react`/`react-dom` 18.3.x, `vite` 5.4.x, `@vitejs/plugin-react` 4.3.x)
after `npm create vite@latest`'s current defaults (Vite 8, a rolldown-based
bundler) failed outright under this machine's Node version (`20.18.0`,
just under the `^20.19.0` the rolldown native bindings need) — a real,
reproducible gotcha, not a one-off.

Structure: `main.jsx` (entry), `App.jsx` (switches between the login view
and a logged-in placeholder; owns logout), `api.js` (backend calls),
`auth.js` (token storage), `pages/LoginPage.jsx`. Wired to the backend per
§1's decisions: bearer token read from `localStorage` on every authenticated
call (Vuln 11), no Vite dev-server proxy (would make requests same-origin
and mask the CORS vulnerability), `VITE_API_BASE_URL` configurable via env
var for the Docker vs. local-dev split.

**Logout was added and isn't in `phase 1 spec.md` at all.** Resolved as
frontend-only — clearing the stored token — since the backend is a
stateless JWT resource server with no session to invalidate; confirmed
explicitly rather than assumed before building it.

Not built: the registration page (Step 3) and webhook page (Step 6) —
deferred, following `phase 1 spec.md`'s own step order rather than building
every page at once.

---

# Week 4 progress — backend endpoint/config changes

Output for requirement 6 of
[`Weekly Requirements/week-04.md`](../Weekly%20Requirements/week-04.md)
(added retroactively).

Implemented in code, each tagged per `phase 1 spec.md`'s numbered-comment
convention:
- **Vuln 1** (permissive CORS) — `CorsConfigurationSource` bean in
  `SecurityConfig.java`.
- **Vuln 2** (verbose errors) — three `server.error.*` properties in
  `application.properties`.
- **Vuln 3** (CSRF disabled) — pre-existing code, now commented; confirmed
  non-exploitable in practice given Vuln 11's header-based (not cookie)
  token transport, and that reasoning is recorded inline at the code site.
- **Vuln 4** (exposed Actuator) — `spring-boot-starter-actuator` dependency,
  `management.endpoints.web.exposure.include=*`, and a matching
  `.requestMatchers("/actuator/**").permitAll()` rule — both touchpoints
  tagged with the same vuln number per the spec's convention for a single
  vulnerability spanning multiple locations.

This completes `phase 1 spec.md` Step 1 in full. Tally: **5 of 15** numbered
vulnerabilities now exist in code (1, 2, 3, 4, 11). Not built: Vulns 5–10
and 12–15 (Steps 2–6 — database baseline, registration, login's backend
half, the fuzz test, webhook registration).

---

# Week 4 progress — CI wiring: SonarQube + setup guide

Output for requirement 7 of
[`Weekly Requirements/week-04.md`](../Weekly%20Requirements/week-04.md)
(added retroactively).

Added a `sonarqube` job to `.gitlab/ci/static-analysis.yml`, running after
`semgrep` via `needs` rather than in parallel. Real problems hit while
getting it working, each now documented in
[`docs/sonarqube-setup.md`](../sonarqube-setup.md) so they don't have to be
rediscovered:
- `mvnw`'s executable bit wasn't actually set in git's tracked file mode
  (`100644`, not `100755`) despite looking executable locally — fixed both
  the tracked mode and switched the job to `sh mvnw` so it doesn't depend
  on that bit surviving future checkouts.
- `mvn sonar:sonar`'s plugin-prefix resolution didn't work in this
  Maven/image combination — fixed by using the plugin's fully-qualified
  coordinates instead of the shorthand.
- SonarCloud assumes a `master` main branch by default; this repo's actual
  branch (`feature/test-stat-ci`) had to be set as the project's main
  branch manually in SonarCloud's UI for the dashboard to show anything.
- **The significant one:** SonarQube Cloud's Free plan doesn't support
  custom Quality Gates, only the built-in "Sonar way" gate — which is
  almost entirely New-Code-scoped, so it would pass trivially regardless of
  how many long-standing seeded vulnerabilities exist in the codebase.
  Worked around by bypassing the Quality Gate mechanism entirely: the job
  polls SonarCloud's Compute Engine task for completion, then queries the
  Issues Search API directly for every unresolved issue on the branch
  (not just New Code), prints them to the job log (closing the "why
  doesn't this look like Semgrep's inline output" gap raised earlier), and
  fails the job only on `BLOCKER`/`CRITICAL` severity.

`docs/sonarqube-setup.md` was written as a generic, second-person guide for
students mirroring this repo with their own SonarQube Cloud account —
deliberately scrubbed of first-person references to this project's own
setup experience, per explicit direction, so it reads as documentation for
the audience rather than a narrated postmortem.

---

# Week 4 progress — Docker hot-reload infrastructure

Output for requirement 8 of
[`Weekly Requirements/week-04.md`](../Weekly%20Requirements/week-04.md)
(added retroactively).

**Backend:** switched `docker-compose.yml`'s `backend` service from
building/running `Dockerfile`'s packaged-jar image to a dev-mode setup —
`Dockerfile.dev`, source bind-mounted, `dev-entrypoint.sh` running
`mvn spring-boot:run` in the background while polling for source changes
and triggering `mvn compile` on them (`spring-boot-devtools`, already a
`pom.xml` dependency, then restarts the app automatically once
`target/classes` changes — it reacts to compiled output, not `.java`
source directly, so something has to drive that recompilation).

One real bug hit and fixed: the first version of `dev-entrypoint.sh`
compared source-file timestamps against `target/classes`'s own directory
mtime as the "last compiled" reference point — that directory's timestamp
doesn't reliably advance every time Maven recompiles, so once source files
were newer than that stale reference, the check stayed true forever,
recompiling on every single poll instead of only real changes. Fixed by
using a self-controlled sentinel file (`touch`ed immediately after each
compile) as the reference point instead of relying on Maven's own
directory-mtime side effects.

**Frontend:** `vite.config.js` gained `server.watch.usePolling` (with a
300 ms interval) and `server.hmr.clientPort`, plus a `CHOKIDAR_USEPOLLING`
environment variable on the `frontend` service as a backup — native
filesystem change events don't reliably cross a Docker bind mount on
Windows hosts, and Vite's own config-file watcher has the same problem
watching itself, so a container recreate (not just a file save) is needed
for watch-related config changes to actually take effect.

**Known tradeoff:** `backend/Dockerfile` (the production-style multi-stage
build) is no longer what `docker compose up` runs at all — left in place,
currently unused. Reconciling that (retire it, or keep it for a future
non-dev build path) is an open decision, not resolved this week.
