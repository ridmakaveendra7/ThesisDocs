# Week 4 requirements

## Goal

Week 3 produced a backend-only vulnerability catalog and fuzz-target
research, explicitly deferring anything frontend-specific because "the
frontend stack isn't decided yet" (`Weekly Progress/week-03.md`). This week
closes that gap: decide a frontend and a database for Phase 1, work out what
new vulnerabilities those choices bring (for both Phase 1 and the full
Phase 2 system), and turn all of that into the final, concrete, detailed
Phase 1 build spec.

This week's deliverable is **research and decisions, not implementation** —
same split as week 3. Output goes in `docs/Weekly Progress/week-04.md` once
done: the frontend decision, the database decision, the vulnerability
mapping, and the concrete Phase 1 spec (see shapes below). Actual code
changes are later work, started only once this spec exists and has been
reviewed.

**Scope note:** this week redefines Phase 1 itself — the staff-built base
app every (hypothetical) student inherits — to potentially include a real
frontend and a deliberately chosen database, not just a bare JSON API. Any
decision made here should still be justified as a good default for a
*template/example* students build on, not as whatever's fastest to ship
once — see `finsec_app_final_spec.md`'s framing of Phase 1 as staff-built
and Phase 2 as student-built for the standard this needs to meet.

---

## 1. Frontend decision

### Goal

Decide which frontend stack Phase 1 ships with. Nothing has committed to
one yet — week 3's research explicitly scoped frontend-specific concerns
out "until a frontend exists."

### What to produce

A short comparison (not necessarily a big table) covering at least:
- The realistic candidate stacks — React, Vue, Angular, Svelte, a
  no-framework vanilla JS/HTML approach, and a server-rendered option
  (e.g. Thymeleaf inside Spring Boot, avoiding a separate frontend
  entirely) — each judged, not just listed.
- A clear final choice with the reasoning that drove it.
- What the choice implies for things week 3 left open specifically because
  no frontend existed: where the auth token gets stored client-side
  (`localStorage` vs. cookie) and what that does to the already-seeded
  CSRF-disabled finding's real exploitability; whether a CSP gets added;
  how the existing CORS misconfiguration's exploitability changes once a
  real frontend origin exists.

### Judging criteria

- **Fits the "minimum base code" constraint** that shaped every week-3
  decision — a teaching template shouldn't carry a heavier frontend than a
  ~5-page app needs.
- **Security-tooling coverage** — does Semgrep/ESLint/SAST tooling actually
  have rules for this stack, consistent with the tool-detectability
  emphasis the whole project has had since week 3.
- **Escape-by-default vs. escape-optional templating** — whether a DOM-XSS
  finding on this stack would look like a believable, deliberate mistake
  (an opt-out from a safe default) or a structurally forced outcome
  (unescaped by default) — matches the "vulnerability must be an
  intentional choice" discipline used for every backend finding so far.
- **Fits the project's own architecture** — `finsec_app_final_spec.md` §2
  models two separate clients (app + web) calling one JSON backend across
  trust boundaries A/B; consider whether the frontend choice should reflect
  that shape even at Phase 1 scale.
- **Realistic for a fintech app** — not an invented demo shape, consistent
  with how every backend vulnerability carrier was chosen (week 3's
  "thematically plausible" criterion).

### Non-goals

- No requirement to build anything this week — decision and reasoning only.
- No obligation to lock in Phase 2's web client to the same stack —
  `finsec_app_final_spec.md` §3.3 already leaves that as a free,
  justification-graded student choice; confirm whether this week's decision
  changes that framing or leaves it alone.

---

## 2. Database decision

### Goal

Decide which database Phase 1 (and, by extension, the system this app is
heading toward) should run on. **Disregard `finapp/`'s existing Postgres +
`docker-compose` setup as a given** — re-derive the choice from first
principles; "it's already there" is not by itself a justification.

### What to produce

A short comparison covering at least PostgreSQL, MySQL/MariaDB, embedded H2,
SQLite, and a NoSQL option (e.g. MongoDB), each judged against the criteria
below, with a clear final choice and reasoning — including whether it
happens to match or differ from what's currently running.

### Judging criteria

- **Fits the banking domain** — Phase 2 heads toward accounts,
  transactions, payments, and derived metrics (`finsec_app_final_spec.md`
  §5.4); consider what data-integrity guarantees that domain actually needs
  (relations, transactional writes) before defaulting to whatever's
  familiar.
- **Infra-security teaching value** — this thesis's focus is CI/CD security
  evidence; consider whether an embedded/in-memory DB hides infra-level
  concerns (externalized credentials, a network-reachable DB container)
  that a real containerized DB would expose to the exercise.
- **Tooling/ecosystem maturity** — Spring Data JPA support, availability of
  security-hardening guidance to cite.
- **Local dev/test cost** — whether the choice requires Docker for every
  `mvn test` run, and whether a lighter fallback (e.g. an embedded DB for
  tests only) is worth adding alongside the primary choice.

### Non-goals

- No requirement to migrate or touch `finapp/`'s actual configuration this
  week — decision and reasoning only.
- No requirement to decide Phase 2's eventual schema (accounts,
  transactions, payments tables) — just the engine.

---

## 3. Vulnerabilities the frontend and database choices bring

### Goal

Week 3's vulnerability catalog is backend-only by its own scope note. Now
that a frontend and a database are being decided, work out what *new*
vulnerability candidates those choices themselves introduce or enable —
separately for Phase 1 (what the base app can actually carry) and Phase 2
(what only becomes possible once the full system — bank integration,
payments, the access matrix, derived metrics — exists).

### What to produce

Two catalogs (frontend-sourced, database-sourced), each split into a Phase 1
list and a Phase 2 list, following the same catalog shape week 3 used:
category/CWE, carrier, why it's realistic here, and — critically — why it
belongs in Phase 1 vs. Phase 2 specifically (i.e. what feature or decision
has to exist first for the candidate to be real rather than hypothetical).

### Candidate areas to investigate

Starting checklist, not a final list — confirm each is actually realistic
given whatever frontend/database gets chosen above before including it.

**Frontend-sourced, plausible even at Phase 1 scale:**
- DOM-based XSS, distinct from the already-seeded backend reflected-XSS —
  specifically whether the frontend's own rendering of a backend response
  (or any client-side templating) reintroduces or chains with it.
- Where the auth token lives client-side and what reads it — ties directly
  into the frontend decision above.
- Missing/absent security headers now that there's a real page for a DAST
  tool to check (CSP, clickjacking-relevant headers) — week 3 already
  flagged "missing security headers" as a backend-config candidate; check
  whether a frontend changes what's actually missing.
- Whether the existing CORS misconfiguration's exploitability changes once
  a real frontend origin exists to demonstrate it from.
- Re-confirm whether the CSRF-disabled finding's exploitability changes
  based on the frontend's token-transport choice (cookie vs. header).

**Frontend-sourced, plausible only once Phase 2 features exist:**
- Rendering of free-text bank data (`creditorName`,
  `remittanceInformationUnstructured` — already flagged in
  `finsec_app_final_spec.md` §5.6 as needing output-encoding).
- Client-side UI decisions that could be mistaken for the actual
  server-side access-matrix enforcement §5.3 requires.
- Handling of the bank SCA redirect flow (`state`, `scaRedirect`) on the
  client side.
- Anything resembling clickjacking on a real money-moving screen (payment
  approval), which doesn't exist until Phase 2.
- Any embedded/cross-origin messaging (`postMessage` or similar) introduced
  by a bank-flow integration.

**Database-sourced, plausible even at Phase 1 scale:**
- The already-seeded SQL injection finding (week-3 catalog §6) — worth
  restating explicitly as a DB-choice-driven finding, not just a backend
  one.
- Whether the DB credential-handling story (currently only in a gitignored
  `.env`) has a realistic "committed to a scanned file" variant worth
  seeding, parallel to the already-planned hardcoded-JWT-secret carrier.
- Whether the `docker-compose` network topology has any realistic
  misconfiguration worth flagging (e.g. an exposed DB port) even if not
  currently present.
- DB user privilege scope, at Phase 1's current single-schema size.

**Database-sourced, plausible only once Phase 2 data exists:**
- Storage protection for bank access/refresh tokens and `consentId`
  (`finsec_app_final_spec.md` §5.4 requires server-side custody — check
  what "custody" implies for storage protection, not just network
  exposure).
- Whether new native/raw queries Phase 2 features are likely to add repeat
  the SQL injection lesson with real financial data at stake.
- Whether the derived metrics (§5.4) — already flagged there as creating a
  GDPR-relevant behavioural profile — have a *technical* storage-protection
  angle worth adding to the *legal* one already documented.
- Transactional integrity on money-movement writes (payment initiation, the
  cumulative low-value-payment limit counting in §5.3).
- Backup/snapshot handling, once there's real data worth backing up.

### Non-goals

- No requirement to seed every candidate as actual Phase 1 build work —
  most of this, especially the Phase 2 list, is for later phases; this
  week is about identifying and justifying candidates, same as week 3's
  vulnerability research was.
- Don't re-litigate or re-catalog anything already in week 3's
  backend-only catalog — only what's *newly* introduced by the frontend/DB
  decisions.

---

## 4. Final concrete Phase 1 spec

### Goal

Once §1–3 above are decided, turn them into an updated, concrete, detailed
Phase 1 build spec — the actual thing implementation would start from.

### What to produce

An update to (or clearly-scoped extension of) `working_spec_finsec.md`
reflecting:
- The chosen frontend and what it adds to Phase 1's surface (pages/views,
  how each maps to an existing backend endpoint).
- The chosen database and any changes that implies for the existing
  `docker-compose`/properties setup — or confirmation that none are needed.
- Which of the new vulnerability candidates from §3 are actually selected
  as Phase 1 build targets (vs. deferred to Phase 2, same treatment week
  3's catalog gave XXE, weak randomness, etc.).
- An updated verification checklist covering whatever's newly in scope.

### Non-goals

- Don't start implementation from this spec this week — per the Goal
  section above, this week ends at a reviewed spec, not code.
- Don't assume the frontend/database decisions from §1–2 without actually
  producing the reasoning — the concrete spec should read as a consequence
  of that research, not a restatement of a foregone conclusion.

---

## 5–8: retroactively added implementation scope

**These four sections were written *after* the work they describe had
already happened, not before** — the opposite order this repo's own
convention calls for (`docs/README.md`: requirements written before a
week's work starts). §1–4 above were decided and followed properly; §5–8
are being added now to make the record honest about what actually got
built this week, not to claim it was planned in advance. See
`Weekly Progress/week-04.md`'s matching sections for what was actually
delivered against each.

### 5. Frontend scaffolding (implementation)

**Goal:** build the actual frontend app §1 decided on and §4's spec lays
out — not just the framework decision, but a working app.

**What to produce:** a real React + Vite project at `finapp/frontend/`; the
login page/flow (`phase 1 spec.md` Step 4's frontend half); wired to the
backend per §1's CORS and token-transport decisions.

**Non-goals:** registration, webhook, and any other later-step page —
those follow `phase 1 spec.md`'s own step order, not all-at-once.

### 6. Backend endpoint/config changes (implementation)

**Goal:** implement the vulnerabilities `phase 1 spec.md` Steps 0–1
specify, in actual code, not just as a documented plan.

**What to produce:** Vuln 1 (permissive CORS), Vuln 2 (verbose errors),
Vuln 3 (CSRF-disabled, tagged), Vuln 4 (exposed Actuator) present in code,
each carrying the numbered-comment convention `phase 1 spec.md` defines.

**Non-goals:** Steps 2–6 (database baseline, registration, login's backend
half, the Jazzer fuzz test, webhook registration) — not this week.

### 7. CI wiring: SonarQube integration + setup guide

**Goal:** extend `.gitlab/ci/static-analysis.yml` beyond Semgrep (still
open work from week 3 / `finsec_app_final_spec.md` §5.7), and make that
extension reproducible for students who mirror this repo with their own
SonarQube account rather than this project's.

**What to produce:** a `sonarqube` job in `.gitlab/ci/static-analysis.yml`,
including working around the Free-tier Quality Gate limitation (a
custom Issues-API-based check instead); `docs/sonarqube-setup.md` as a
student-facing setup guide.

**Non-goals:** OWASP ZAP wiring — still open, not this week.

### 8. Docker hot-reload infrastructure

**Goal:** make local iteration on the above practical — edits to backend or
frontend source should be reflected without a manual image rebuild, since
that's the normal way development actually happens against this stack now.

**What to produce:** a dev-mode `backend` service (`Dockerfile.dev`,
`dev-entrypoint.sh`, Maven + bind mount + `spring-boot-devtools`-driven
restart) and a dev-mode `frontend` service (Vite + polling-based file
watching) in `docker-compose.yml`.

**Non-goals:** a production-shaped deployment story — `backend/Dockerfile`'s
packaged-jar build is left in place but is no longer what
`docker compose up` actually runs; reconciling that is a later decision,
not this week's.

---

## Non-goals for this week (overall)

- No Phase 2 feature work (bank integration, device-key auth, access
  matrix, derived metrics) — this week only reasons about what Phase 2
  *would* introduce, for the vulnerability-mapping task in §3, and §5–8's
  implementation work stays within Phase 1's own scope.

## Open questions to resolve during research

- Does the frontend decision (§1) change anything about how Phase 2's web
  client is scoped in `finsec_app_final_spec.md` §3.3 (currently: free,
  justification-graded student choice), or are these fully independent
  decisions?
- Are any of the new database-vulnerability candidates in §3 good enough
  (yield, detectability, thematic plausibility — same criteria week 3 used)
  to actually become Phase 1 build targets, or do they all belong in the
  Phase-2-only list?
- Does adding a frontend change anything about which CI tools/stages are
  needed beyond what week 3 already flagged as open (SonarQube/ZAP wiring,
  `finsec_app_final_spec.md` §5.7)?
