# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout

This working directory is the container for a Master's thesis ("Secure Software
Development", Fulda University of Applied Sciences), not itself a git repo. The
actual codebase lives one level down:

- `finapp/` — the only git repository here (remote: GitLab at
  `git-ce.rwth-aachen.de/.../secure-software-pipeline-demo`), currently on
  branch `feature/test-stat-ci`. All development work happens here. As of
  Phase 1's frontend work, it's a multi-service layout, not a single Spring
  Boot project at its root:
  - `finapp/docker-compose.yml`, `finapp/.env` — orchestrate the whole
    stack (`backend` + `db` + `frontend` services); kept at the repo root
    since Docker Compose auto-loads `.env` from the compose file's own
    directory.
  - `finapp/backend/` — the Spring Boot app (moved here from the repo root
    during Phase 1 restructuring; every path below that used to say
    `finapp/src/...` etc. is now `finapp/backend/src/...`).
  - `finapp/frontend/` — the React + Vite SPA (Phase 1's frontend, per
    `phase 1 spec.md`).
- `docs/Weekly Requirements/` — one file per week (week-numbered), each stating
  that week's planned work. **Defining these requirements week by week is
  itself part of the thesis** — always check the highest-numbered file here
  for what's currently planned; don't assume earlier weeks' scope still holds.
- `docs/Weekly Progress/` — one file per week (week-numbered), written after
  that week's work is done, recording what actually happened. Check the
  highest-numbered file here (and the code itself) for the true current state
  of `finapp/` — don't assume a requirement was fully/correctly implemented
  just because it was planned.
- `temp_FinSec_Project_Specification_EN.md` — a **reference-only** brief from a
  related course exercise describing one possible FinSec design. It is
  inspiration/terminology, not a spec to implement literally — don't treat it
  as authoritative or assume `finapp/` needs to match it, and don't confuse it
  with the actual weekly requirements above.
- `runner/gitlab-runner.exe` — a local GitLab Runner binary, unrelated to app code.
- Two PDFs (thesis submission, thesis proposal) — reference material, not inputs
  to code changes.

## Current state and requirements

Current planned work and actual state live in the weekly, week-numbered files
under `docs/Weekly Requirements/` and `docs/Weekly Progress/` (see above) —
always read the latest week in each, not just week 1, and don't rely on this
file's Architecture section below staying in sync with them as weeks progress.

## Commands (run from `finapp/backend/`, not `finapp/`)

Build requires **JDK 17+** (`pom.xml` targets `java.version=17`; Spring Boot
4.1.0's baseline also requires 17+). The system default `JAVA_HOME` on this
machine points at a JDK 8, which will fail the build — override it per-command,
e.g. pointing at the Corretto/Adoptium 17 install already present on this
machine:

```powershell
$env:JAVA_HOME = "C:\Users\ridma\.jdks\corretto-17.0.13"
.\mvnw.cmd clean package
```

```bash
JAVA_HOME="/c/Users/ridma/.jdks/corretto-17.0.13" ./mvnw clean package
```

- Build: `./mvnw clean package`
- Run tests: `./mvnw test`
- Run a single test class: `./mvnw test -Dtest=FinappApplicationTests`
- Run the app locally: `./mvnw spring-boot:run` (no active profile is set by
  default — see Profiles below; without one it uses the default
  `application.properties`, which has no datasource configured)
- Docker (backend + Postgres + frontend, run from `finapp/`, not
  `finapp/backend/`): `docker compose up --build` — reads `POSTGRES_*`,
  `DB_*`, `JWT_SECRET` from `.env` (present locally, gitignored, not
  committed). Per `phase 1 spec.md`, this is the standard way to run the
  whole stack during Phase 1 implementation — there's no H2/embedded-database
  fallback, so `./mvnw test` from `finapp/backend/` also needs the `db`
  container running (`docker compose up -d db` from `finapp/`).

## Architecture

Standard Spring Boot layering under `src/main/java/com/sec/finapp/`:
`controller/` → `service/` → `repository/` (Spring Data JPA) → `model/`
(JPA entities), with request/response POJOs in `dto/`.

- **Auth flow**: `AuthController` (`/api/auth/register`, `/api/auth/login`) is
  the only implemented API surface (the earlier `hello`/`/api/hello` probe
  endpoint was removed — see `phase 1 spec.md`'s Step 0 for why no
  throwaway ping endpoint is used for frontend/backend connectivity
  checking instead). `AuthService` hashes passwords with BCrypt and persists `User`
  (username, password hash, a flat `role` string). `UserService` implements
  Spring Security's `UserDetailsService` by loading a `User` and wrapping it as
  a Spring Security `UserDetails`.
- **JWT**: `SecurityConfig` wires the app as an OAuth2 **resource server**
  validating its own HMAC-signed JWTs — it builds a `SecretKey` from the
  base64 `jwt.secret` property and uses the same key for both `JwtEncoder` and
  `JwtDecoder` (self-issued/self-verified tokens, not a third-party IdP).
  `JwtService.generateToken` issues 1-hour tokens carrying a space-joined
  `roles` claim. `/api/auth/**` is `permitAll()`; every other request must
  present a valid bearer JWT (`anyRequest().authenticated()`). CSRF is
  disabled (stateless JWT API).
- **Profiles**: `application.properties` has no datasource and no active
  profile — it's meant to be combined with `dev` (H2-style local dev, SQL
  logging on, security DEBUG logging), `docker` (reads `DB_URL`/`DB_USERNAME`/
  `DB_PASSWORD` env vars, matches `docker-compose.yml`), or `prod` (currently
  an empty file — not yet configured). Pick a profile explicitly via
  `-Dspring-boot.run.profiles=dev` or `SPRING_PROFILES_ACTIVE`.
- **Bootstrap data**: `FinappApplication` seeds a hardcoded user
  (`ridma`/`password123`) via `CommandLineRunner` if not already present — a
  dev convenience, not something to rely on in tests or ship to prod.
- **CI**: `.gitlab-ci.yml` has a single `security` stage that triggers
  `.gitlab/ci/static-analysis.yml`, which runs `semgrep scan --config p/default
  --error` in a `python:3.12-slim` image. This is the "automated security
  evidence" pipeline the thesis spec calls for — currently just Semgrep SAST,
  nothing else wired in yet.

## Known repo quirks

- `finapp/default.yaml` is a ~2.3 MB dump of Semgrep's default ruleset
  (`p/default`), currently staged in git (`git status` shows it as `A`). It's
  almost certainly a leftover from running `semgrep` locally, not something to
  read for app config — don't confuse it with a Spring `application.yaml`.
