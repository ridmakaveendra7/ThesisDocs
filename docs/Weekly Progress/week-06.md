# Week 6 progress

**Author:** Ridma Kaveendra

There is no `Weekly Requirements/week-06.md`; this week followed the "Next
week" list in [`week-05.md`](week-05.md): a deep dive into the ZAP job, an
SBOM stage, and the open ZAP questions (does the browser login work, does the
active scan find the SQL injection).

## What was done

### Backend

- **Build fix:** a new `hello.java` declared `public class helloController`,
  which Java rejects (file and class name must match). That single error
  stopped Lombok from running, so the build also reported 17 unrelated
  "cannot find symbol" errors for getters, setters, `log` and constructors.
  Renamed to `HelloController.java`.
- **`GET /api/hello`** (authenticated) is back as a simple protected endpoint.
  The frontend and ZAP use it to check that a login really worked. Note:
  `phase 1 spec.md` Step 0 removed this endpoint on purpose, so the spec
  should be updated to match.
- **OpenAPI / Swagger UI:** added `springdoc-openapi-starter-webmvc-ui` (and
  removed the now-duplicate `-api` starter). `/v3/api-docs`,
  `/swagger-ui/**` and `/error` are `permitAll()`. Permitting `/error` matters:
  without it every 404 or 500 was shown as a 401, which is what made Swagger
  UI look "blocked" when it was really just not loaded yet.

### Frontend

- **Routing (react-router-dom 7):** `/login` and `/register` are only shown to
  logged-out users, `/` only to logged-in users; everything else redirects.
  The logged-in placeholder moved to `HomePage.jsx`.
- **Login state:** `isAuthenticated()` checks that a token exists *and* its
  `exp` has not passed, and removes expired tokens.
- **Token on every request:** `api.js` now sends every backend call through
  one `apiFetch` wrapper that adds `Authorization: Bearer <token>` when one is
  stored. A 401 on a request that carried a token logs the user out and
  redirects to `/login`; a 401 without a token (wrong password) is shown as a
  normal error. The home page calls `/api/hello` through it.
- **Backend address:** no longer hardcoded to `localhost:8080`. If
  `VITE_API_BASE_URL` isn't set, the app uses the hostname the page was loaded
  from plus port 8080. This was needed for ZAP (see challenges).
- **Production image:** new `nginx.conf` with an SPA fallback (without it,
  opening `/login` directly returns 404), and the Dockerfile no longer bakes in
  `localhost:8080`.

### ZAP: authenticated scanning, locally

A local ZAP Automation Framework plan (`zap/zap.yaml`) now scans both parts of
the app, each logged in:

- **Backend:** imports every endpoint from `/v3/api-docs`, logs in with a JSON
  request to `/api/auth/login`, sends the token as a header, and runs an active
  scan.
- **Frontend:** ZAP's browser logs in through the real login form
  (browser-based authentication), then crawls the logged-in app with the
  client spider. Login is verified by polling `/api/hello` every 5 minutes.
- **Results:** the active scan confirmed **Vuln 9 (SQL injection)**: ZAP's
  time-based payloads (`pg_sleep(15)`) ended up inside the SQL statement
  Postgres executed. ZAP also reported "HTTP Only Site" (no HTTPS), which is
  why each target appears twice in the report (`http://` and `https://`).

### ZAP: CI job

- New `.gitlab/ci/zap-ci-test.yml` plus the plan `zap/zap-ci.yaml`. The job
  builds `docker-compose.ci.yml` inside Docker-in-Docker, waits for both
  services, registers the test user (the CI database starts empty), then runs
  the official ZAP image on the stack's network (`finapp-network`), where the
  services are reached as `http://frontend` and `http://backend:8080`.
- The report (including its styling folder), ZAP's log and the app's logs are
  kept as artifacts.
- `docker-compose.ci.yml` fixes: frontend port `80:80` (nginx listens on 80,
  the old `5173:5173` pointed at nothing), and the frontend is built with
  `VITE_API_BASE_URL=http://backend:8080` so ZAP's browser can reach the API.
- The CI report covers both the frontend and the backend.

### SBOM and vulnerability scanning

New child pipeline `.gitlab/ci/sbom-ci.yml`:

- **Stage `sbom` (Syft v1.54.0, CycloneDX JSON):**
  - `backend-sbom.cdx.json` is generated from the built JAR, not `pom.xml`,
    because `pom.xml` leaves most versions to the Spring Boot parent. Result:
    104 libraries, all with exact versions.
  - `frontend-sbom.cdx.json` is generated from `package-lock.json`. Result:
    10 libraries (React, React Router and their dependencies); build-only tools
    like Vite are excluded by default.
  - Both ran successfully in CI.
- **Stage `vulnerability-scan` (Grype v0.119.0):** each SBOM is scanned only
  after its own SBOM job succeeds (`needs`). The table goes to the log, the
  JSON report is kept as an artifact, and the job fails on High or worse
  (`GRYPE_FAIL_ON: high`). Locally: **backend 25 findings** (3 Critical,
  11 High, 11 Medium: Tomcat 11.0.22, both Jackson 2.x and 3.x, the PostgreSQL
  driver, log4j-api), **frontend 0**. Every finding has a fixed version, and
  most come from the versions Spring Boot 4.1.0 brings in.
- The SBOM also showed that **two Jackson versions (2.21.4 and 3.1.4)** are
  both packaged.
- **Cosign** was tried locally (v3.1.3): key pair, `sign-blob` on the SBOM,
  `verify-blob`, and a tamper test (changing one version in the SBOM made
  verification fail). Not in CI yet.

## Challenges

- **"context is null" in ZAP.** Every failed frontend login made ZAP's
  *Insights* feature shut the scan down at a 100% login-failure rate, which
  discards the whole session, including the backend context the next job
  needed. Putting the backend jobs first only hid the problem.
- **ZAP's browser logged in to ZAP itself.** The frontend called
  `localhost:8080`; inside the ZAP container that is ZAP's own API, which
  answered "Bad Format". Fixed with the hostname-based backend address. The
  old value also survived in the running container until it was recreated
  (`docker compose up -d --force-recreate frontend`), because containers keep
  the environment they were created with.
- **Frontend login still failed for three more reasons,** found one at a time
  with ZAP's authentication diagnostics report:
  - the `zap-test` user didn't exist (401);
  - the backend wasn't in the frontend context, so ZAP's own login check was
    sent without the token;
  - a final `WAIT` step made ZAP press Enter on a password field that had
    already disappeared after login ("stale element").
- **PowerShell strips double quotes** from arguments passed to `docker`/`curl`,
  so a JSON body arrived as `{username:...}` (400). `Invoke-RestMethod` or
  escaped quotes avoid it.
- **Slow scans:** ZAP logged in again before every active-scan rule (about
  20 s per browser login). Switching verification to polling `/api/hello`
  reduced this.
- **Dev containers crashing mid-scan:** Docker Desktop's Windows file sharing
  briefly failed (`EIO: i/o error`). Vite and Spring DevTools read that as
  "every file changed" (false HMR updates and restarts), then the frontend
  crashed. The backend's `dev-entrypoint.sh` exits on any failed command
  (`set -e`), which stopped the backend the same way. One later local scan
  "succeeded" with no frontend results because the frontend container was
  still down. Scanning the built images (CI stack) avoids this.
- **GitLab:** pipelines failed to start when `.gitlab-ci.yml` referenced a CI
  file that wasn't committed yet. At the end of the week, pushes stopped
  creating pipelines at all and a merge request stayed at "almost ready",
  which points to a delay on the GitLab server, not the configuration. The
  local runner also logged DNS errors (`no such host`) after the PC slept.
- **My own mistakes:** a test build left a stray `frontend/C:/...` folder that
  got staged in git (removed), and the first CI artifact list missed the
  report's styling folder (fixed).

## Still open

- Step 6: the webhook feature and Vuln 12 (SSRF).
- Grype in CI hasn't run yet because of the GitLab problem above. Once it
  does, the backend job will **fail** on the current findings; decide whether
  that is wanted, should be `allow_failure`, or whether to upgrade Spring Boot.
- Cosign isn't in CI yet (key handling, or keyless signing).
- The old `.gitlab/ci/zap-ci.yml` is truncated to a single line in the working
  tree; restore it or delete it now that `zap-ci-test.yml` replaces it.
- Gitleaks stage (planned last week) not started.
- `dev-entrypoint.sh` should tolerate a failed file check, and the dev
  services could use `restart: unless-stopped`.
- The parallel-pipeline question from last week (`concurrent = 1`) is still
  unconfirmed.

## Next week

- Confirm the SBOM + Grype pipeline in CI and decide the failure policy.
- Add Cosign signing and verification of the SBOMs to the pipeline, verified
  before Grype scans them.
- Add the Gitleaks stage.
- Step 6 (webhook, Vuln 12).
