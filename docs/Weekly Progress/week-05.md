# Week 5 progress

**Author:** Ridma Kaveendra

Output for [`Weekly Requirements/week-05.md`](../Weekly%20Requirements/week-05.md):
finishing Phase 1's seeded vulnerabilities, plus the CI work that grew out of
it (fuzzing and DAST).

## What was done

### Phase 1 vulnerabilities (Steps 2–5)

14 of the 15 numbered vulnerabilities in [`phase 1 spec.md`](../../phase%201%20spec.md)
now exist in code, each tagged with its `Vuln N` comment. Only Vuln 12 (SSRF via
the webhook, Step 6) is still missing, together with the webhook feature itself.

- **Step 2, database baseline:** Vuln 5 (committed DB password fallback in
  `application-docker.properties`), plus the Vuln 14/15 guardrail annotations
  in `docker-compose.yml`.
- **Step 3, registration:** Vuln 6 (backend returns the username unescaped in
  an HTML confirmation), Vuln 7 (new registration page renders it with
  `dangerouslySetInnerHTML`), Vuln 13 (missing-CSP annotation in `index.html`).
- **Step 4, login backend:** Vuln 8 (hardcoded JWT secret; the now-unused
  `JWT_SECRET` config was removed), Vuln 9 (login lookup builds its SQL by
  string concatenation), Vuln 10 (unescaped login audit logging).
- **Step 5, fuzz testing:** `JwtDecoderFuzzTest` against JWT validation, as the
  spec requires.

### Fuzz testing beyond the spec

- **`UserServiceFuzzTest`** fuzzes the login lookup (Vuln 9). It found a
  genuine bug on its own: a username containing a quote or control characters
  breaks the SQL, and the resulting `JDBCConnectionException` is never caught,
  so login crashes with a 500 instead of rejecting the user.
- **Seed corpus:** hand-written starting inputs for that test
  (`O'Brien`, `alice`, `admin'--`) under `src/test/resources/...Inputs/`. This
  needed the test to use `FuzzedDataProvider`, because Jazzer stores inputs for
  typed `String` parameters in its own binary format, which can't be written by
  hand.

### CI pipeline

The pipeline now runs three child pipelines in parallel on every push:

- **Static analysis:** Semgrep and SonarQube (unchanged, now in parallel
  instead of one after the other).
- **Fuzzing (`jazzer-ci.yml`):** finds every `*FuzzTest` class automatically,
  runs each as its own Maven call with a Postgres service, and prints a
  pass/fail summary at the end.
- **DAST (`zap-ci.yml`):** builds the whole app from a new CI-only
  `docker-compose.ci.yml` inside Docker-in-Docker, then scans it with the
  official ZAP image. The scan is authenticated in two ways:
  - **API:** the backend now publishes an OpenAPI spec (springdoc). CI logs in
    as a throwaway test user and ZAP imports every endpoint, sends the token,
    and runs an active scan.
  - **Frontend:** ZAP's browser signs in through the real login form, then
    crawls the app.

  A local test run confirmed the active scan fires SQL-injection and XSS
  payloads at the login endpoint and triggers SQL errors in the backend. ZAP
  also flagged the missing CSP (Vuln 13).
- New supporting files: `frontend/Dockerfile` (build + nginx), and
  `docker-compose.ci.yml`.

### Documentation

- Created `Weekly Requirements/week-05.md`.
- Compared the course schedule against the vulnerability map. IDOR was already
  covered in Phase 2 but not named, so the three affected rows in
  `finsec_feature_vuln_map.md` now say "IDOR, CWE-639".

## Challenges

- **Jazzer only fuzzes one test per Maven run.** When several `@FuzzTest`
  classes run together, all but one are silently skipped. Fixed by running
  each class separately.
- **Jazzer flagged its own database connection as SSRF.** Its built-in SSRF
  check treats every network connection as suspicious, including Postgres.
  Fixed by explicitly allowing `db:5432` inside the fuzz test.
- **DB-backed tests can't run from the host.** The database is only reachable
  inside the Docker network (the Vuln 14 guardrail keeps its port closed), so
  those tests run inside the backend container or in CI.
- **The local GitLab runner kept blocking the ZAP job.** It had
  `privileged = false`, so Docker-in-Docker jobs were never scheduled, and it
  also hung at times and stopped picking up jobs (cancel + retry worked
  around it). Privileged mode was eventually enabled.
- **The stage name `dast` broke the pipeline** with a generic "yaml invalid"
  error. It clashes with GitLab's built-in DAST feature; renaming the stage
  fixed it.
- **A YAML gotcha, twice.** Lines like `- echo "Step 1: ..."` are read as a
  key/value pair because of the colon. They have to be quoted.
- **ZAP never exits after its scan.** The report is written, but the process
  keeps running, which left jobs hanging or killed by timeouts. The job now
  waits for ZAP's "Automation plan" log line instead of for the process to
  stop.
- **ZAP's default port (8080) collided with the backend,** and **ZAP's
  browser crawl found nothing** in the first setup because that ZAP install
  had no browser. Both went away by switching to the official ZAP image.

## Still open

- Step 6: the webhook feature and Vuln 12 (SSRF).
- Not yet confirmed in CI: whether ZAP's browser login succeeds, and whether
  the active scan reports the SQL injection within its 15-minute limit.
- ZAP passes as long as the scan completes, even with findings, unlike Semgrep
  and SonarQube, which fail the pipeline.

## Next week

- Deep dive into the ZAP CI job: understand each part of the scan and its
  report, confirm the open points above, and decide which findings should fail
  the pipeline.
- Add an SBOM stage (software bill of materials for backend and frontend).
- Add a Gitleaks stage to scan the repository for committed secrets.
- Check why the three pipelines don't actually run in parallel. They are set
  up to, but they run one after another in a different order each time. A
  likely cause is the runner's `concurrent = 1` setting in
  `C:\GitLab-Runner\config.toml`, which lets it run only one job at a time.
