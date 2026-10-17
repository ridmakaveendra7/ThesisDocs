# Week 8 progress

**Author:** Ridma Kaveendra

There is no `Weekly Requirements/week-08.md`. This week went into the
student-facing **setup guides** in `finapp/docs/`: guides that let students
rebuild the project's stack and CI pipeline in their own GitLab project. Each
guide was checked by following it step by step in a new, separate project
(`test_app`, its own GitLab repository `test-app`), not just by reading it.
The Phase 1 work planned at the end of week 7 did not move this week.

## What was done

### Setup guides (`finapp/docs/`)

The guides are numbered in the order students follow them:

| # | Guide | Covers | Checked in `test_app` |
|---|---|---|---|
| 01 | `01-backend-frontend-docker-setup.md` | Spring Boot backend, React + Vite frontend, PostgreSQL; development stack with hot reload and production-style images | yes |
| 02 | `02-gitlab-project-runner-setup.md` | GitLab project, first push, own GitLab Runner (Windows, Docker executor), CI/CD variables, test pipeline | yes |
| 03 | `03-semgrep-setup.md` | Semgrep job in a static-analysis child pipeline | yes |
| 04 | `04-sonarqube-setup.md` | SonarQube Cloud job in the same child pipeline | yes |

All guides share one structure: tested versions, prerequisites, numbered
steps, and a pitfalls section taken from problems actually hit in `finapp`
(weekly progress notes) and in `test_app` this week. They are written for
students in general, without references to this repository's own setup or
its deliberate vulnerabilities.

Main content decisions:

- **`master` as the default branch (guide 02).** A SonarQube Cloud project
  created manually starts with a main branch called `master`, and its
  overview only shows that branch. Using `master` in GitLab as well means
  the results appear without changing anything in SonarQube Cloud. The
  guide sets the name locally (`init.defaultBranch`) and explains how to
  check it in GitLab after the first push, including moving an existing
  `main` branch over. Checked: SonarQube Cloud's main branch *can* be
  renamed, so this is a convenience, not a hard requirement.
- **Runner registration (guide 02)** now points to GitLab's own *Register
  runner* page instead of repeating its steps, and keeps only the parts
  GitLab doesn't cover (Docker executor, `config.toml`: `privileged`,
  `concurrent`).
- **SonarQube (guide 04)** was rewritten from a list of fixes into a full
  guide. Its job is the **minimal** version: build the backend and upload
  the analysis, so it passes whenever the upload succeeds and findings are
  reviewed in SonarQube Cloud. `finapp`'s own job additionally waits for
  processing and fails on `BLOCKER`/`CRITICAL` issues through the Issues
  API; the guide leaves that out to keep the first setup simple.
- **SonarQube token (guide 04).** On the Free plan the token is a
  *personal access token*: it acts as the user, with all the user's
  permissions in every organization, not only for one project. The job only
  needs it for uploading (*Execute Analysis*). Project-limited *scoped
  organization tokens* exist only on paid plans, so the guide instead
  requires a masked CI/CD variable, an expiry date, and revoking it if it
  leaks.

### `test_app` (verification project)

A fresh project built only from the guides: backend, frontend, database,
Docker Compose files, GitLab project `test-app` with its own runner, then
the Semgrep and SonarQube jobs (4 commits on 2026-10-17). Both jobs ran
successfully, and the SonarQube results appear in a separate SonarQube Cloud
project. `test_app/` is ignored in `finapp`'s `.gitignore`.

### Clean-up in `finapp`

- `.gitlab/ci/zap-ci-test.yml` deleted (it was identical to `zap-ci.yml`;
  open item from week 7).
- The old `docs/sonarqube-setup.md` replaced by `04-sonarqube-setup.md`.

## Challenges

Each of these is now a pitfall in the guides, because students will hit
them too:

- **Git login to the university GitLab.** The account uses single sign-on,
  so its password is rejected over HTTPS (`HTTP Basic: Access denied`); a
  personal access token with `read_repository`/`write_repository` is needed.
  An SSH remote without a registered key fails differently, by asking for a
  password GitLab never accepts. Pushing to `finapp` was also rejected at
  first while `test-app` worked; a token created inside one project
  (*project access token*) only works for that project, so a **personal**
  access token is needed for several projects.
- **`gitlab-runner` not found.** GitLab's register command assumes the
  runner is on `PATH`, which the Windows download doesn't set up.
- **Runner jobs failing with `failed to connect to the docker API at
  npipe:////./pipe/docker_engine`.** Docker Desktop wasn't running. The
  runner service starts with Windows, Docker Desktop only after login, so
  this can happen after every reboot.
- **Docs that drifted from the code.** The old SonarQube guide referred to
  sections ("main-branch notes", "§5") that no longer existed.

## Still open

Everything listed under "Still open" in [`week-07.md`](week-07.md) is still
open, except the duplicate ZAP file. In particular:

- **Phase 1:** Step 6 (webhook feature + Vuln 12, SSRF) and the
  `phase 1 spec.md` §11 verification checklist.
- **Suppression policy** and the two spec mismatches (manual `fuzz` job,
  `/api/hello`).
- **Setup guides** for the remaining pipeline parts: Gitleaks, ESLint,
  mobsfscan, SBOM + Grype, artefact verification, Jazzer, ZAP.
- **Thesis writing** has not started yet (week 8 of 24).

## Where to start next week

Same priority as at the end of week 7: finish Phase 1 (Step 6, then the
§11 checklist) so it is formally complete. Continue the setup guides in
pipeline order (Gitleaks next), verifying each one in `test_app` as this
week, and start the tool-selection chapter of the thesis from the weekly
progress files.
