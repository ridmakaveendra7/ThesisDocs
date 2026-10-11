# Week 7 progress

**Author:** Ridma Kaveendra

There is no `Weekly Requirements/week-07.md`; this week continued the "Next
week" list in [`week-06.md`](week-06.md): confirm the SBOM + Grype pipeline in
CI, decide its failure policy, and add the Gitleaks stage. It then grew into
an Android app client, two more static-analysis tools, and a first version of
the exposé's "verification of third-party software artefacts" area. The
`feature/zap-ci.dev` branch was merged into `master` (2026-10-08).

## What was done

### SBOM + Grype pipeline

- **SBOM artifacts:** each Grype job now also keeps the SBOM it scanned
  (`<component>-sbom.cdx.json`) next to its report, so one download has both,
  even when the job fails on the severity threshold.
- **Grype ran in CI** and failed the backend job, as expected
  (`GRYPE_FAIL_ON: high`): 4 Critical and 11 High findings in five libraries
  (Tomcat 11.0.22, spring-webmvc 7.0.8, Jackson core/databind 2.x and 3.x,
  PostgreSQL driver 42.7.11), plus 11 Medium. The frontend had none.
- **Fix (commit `089350a`, kept):** Spring Boot 4.1.0 → 4.1.1 (fixes
  spring-webmvc, the PostgreSQL driver and log4j-api), plus version overrides
  for what 4.1.1 doesn't fix yet: `tomcat.version` 11.0.25,
  `jackson-2-bom.version` 2.21.7, `jackson-bom.version` 3.1.7. Verified
  locally that the backend builds and the JAR contains the fixed versions.
  These are third-party library bugs, not the deliberate Phase 1
  vulnerabilities, which are unaffected.

### Pipeline order

The child pipelines now run in a fixed order, one stage each:
static analysis (+ secrets, + artefact verification) → SBOM → fuzz → ZAP.
Before, they all shared one stage and ran in whatever order the runner picked
them up. The later stages use `when: always`, so they still run when an
earlier stage fails (Semgrep and SonarQube fail on the deliberate
vulnerabilities). The ZAP stage is called `zap-scan`, because `dast` clashes
with GitLab's built-in DAST feature (see week 5). All pipelines are enabled
again (the temporary "hide with a dot" testing setup was removed).

### Gitleaks (secret scanning)

- New child pipeline `.gitlab/ci/gitleaks-ci.yml` (Gitleaks v8.30.1), in the
  static-analysis stage. It scans the **whole git history**
  (`GIT_DEPTH: "0"`), since a secret removed in a later commit is still
  readable by anyone who clones the repository. `--redact` keeps secret
  values out of the job log and the JSON report.
- **First run: 3 findings in 70 commits.**
  - 2 false positives: `curl -u "$SONAR_TOKEN:"` in `static-analysis.yml`
    references a CI/CD variable, not a literal token.
  - 1 real finding: the JWT signing key (`jwt.secret=…`) in
    `src/main/resources/application.properties` from the initial commit
    `200ab0dc`, i.e. the file's location before the backend moved to
    `backend/`. The line was removed in `eeea9a2` and the file moved in
    `b98ca0f`, but it stays in the history.
- **Notable result:** this is the same key that is still hardcoded in
  `SecurityConfig.java` as Vuln 8. Gitleaks found it only in the old config
  file in the history, **not** as the `JWT_SECRET = "…"` Java constant in the
  current code: its pattern rules matched the `key=value` config line but not
  the constant. Without the history scan, Vuln 8 would not have been found.
- All three findings are recorded in `.gitleaksignore` (see the note below);
  the job now passes ("no leaks found") but still fails on any new secret.
- Rewriting history to remove the old commit was considered and rejected: the
  key is still in the current code (Vuln 8), so it would not remove the
  secret, while changing every commit ID and requiring a force push.

### Android app client (`appClient/`)

- New Android Studio project (Kotlin + Jetpack Compose, Gradle 9.6, AGP
  9.4.1, `minSdk` 33 / `targetSdk` 37) as the start of the Phase 2 app
  client. `local.properties`, `.gradle/` and `.idea/` are git-ignored.
- **SBOM:** Syft can't read Gradle build files, so Gradle **dependency
  locking** was enabled for `releaseRuntimeClasspath` (the libraries that
  ship in the app). The generated `app/gradle.lockfile` lists 107 libraries
  with exact versions; Syft reads it. New CI jobs `appclient-sbom` (fails if
  the lock file is missing, so an empty SBOM can never look like a clean
  result) and `appclient-grype` (same template and threshold as the others).
  Grype: no known vulnerabilities.
- Semgrep and Gitleaks already covered the folder without any change, since
  they scan the whole repository.

### Static analysis: ESLint and mobsfscan

Both added as jobs in `static-analysis.yml`, next to Semgrep and SonarQube.

- **ESLint (frontend):** ESLint 9 (not 10: `eslint-plugin-react` doesn't
  support it yet) with `eslint-plugin-security` and React's security rules
  (`no-danger`, `jsx-no-script-url`, `jsx-no-target-blank`), config in
  `frontend/eslint.config.js`, run via `npm run lint`. It reports exactly one
  error: `dangerouslySetInnerHTML` in `RegisterPage.jsx`, i.e. **Vuln 7**.
  An `eslint-disable-next-line` suppression was added and then removed again,
  so the job currently **fails** on Vuln 7 on purpose.
- **mobsfscan (app client):** MobSF's source-code scanner (no build, SDK or
  MobSF server needed), mapping findings to OWASP MASVS. It fails on ERROR
  findings only. Results on the template project: 1 ERROR, 1 WARNING
  (`allowBackup="true"`), 6 INFO (missing hardening such as certificate
  pinning and root detection).
  - The ERROR (`android_task_hijacking2`, StrandHogg 2.0) is a **false
    positive**: it only affects apps targeting SDK < 29, and this app targets
    37. mobsfscan reads only `AndroidManifest.xml`, where modern Gradle
    projects no longer declare SDK versions, so it assumed an old one. An
    empty `taskAffinity` didn't clear the rule, and `singleInstance` would
    change app behaviour, so the rule is suppressed in `appClient/.mobsf`
    with the reason (re-check if `minSdk` is ever lowered below 29).
- **Semgrep:** Vuln 9 (SQL injection) is suppressed with a rule-specific
  `nosemgrep` comment on the line Semgrep reports (where the query runs, not
  where the string is built). Semgrep still reports Vuln 4 (Actuator) and
  Vuln 6 (reflected XSS), so its job still fails.

### Verification of third-party software artefacts (exposé area 4)

New child pipeline `.gitlab/ci/artefact-verification.yml` in the
static-analysis stage, plus checks where the artefacts are used:

- **Maven:** `backend/.mvn/maven.config` with `--strict-checksums`, which
  every Maven run reads (Sonar, SBOM, fuzz, and the backend Docker build via
  a new `COPY .mvn`). New job `maven-checksums` downloads every dependency
  and plugin from scratch: **1,211 files checked**. A test with a fake
  repository serving a tampered library showed why it matters: default Maven
  **accepted** it silently, with strict checksums it **refused** it. This
  proves integrity, not the publisher (checksum and file come from the same
  repository).
- **npm:** new job `npm-signatures` (`npm audit signatures`): all **269
  packages** have valid npm registry signatures, **49** also have verified
  Sigstore provenance (which repository and build produced them).
- **Gradle:** `distributionSha256Sum` in the wrapper properties was already
  set by Android Studio and matches Gradle's official checksum.
- **Syft/Grype downloads (tried, then removed):** a script that verified the
  tools before running them (pinned Cosign → Sigstore signature from
  Anchore's release workflow → checksum) worked, and negative tests showed it
  rejects a wrong signer, a modified checksum file and a modified download. It
  was removed again to keep the pipeline simpler, so the CI tools are still
  installed with `curl … | sh`. This is a known, accepted gap.

### Research and design decisions

- **Cryptography in FinSec:** mapped where cryptography appears in Phase 1
  (BCrypt, HMAC-signed JWT with a hardcoded key, no TLS) and Phase 2 (ECDSA
  P-256 device-key signatures, nonces, encrypting bank tokens at rest, OAuth
  `state`/PKCE, refresh-token rotation), the typical mistakes in each, and
  which tool can catch them.
- **App client stack:** a dependency-free Java CLI (`FinsecCli`, as in the
  spec) fits all existing CI tools without new tooling; Android was chosen
  for the app client. Android CI needs no emulator for most checks; fuzzing
  works on plain Kotlin modules with Jazzer, while ZAP only applies to the
  backend endpoints the app calls.
- **Scanning an unknown app-client stack:** Gitleaks and Semgrep work
  without knowing the stack, Syft/Grype work if lock files are committed,
  SonarQube only generically for interpreted languages and Kotlin. Requiring
  lock files in the student spec would make the generic SBOM reliable.
- **Status review against Phase 1 and the exposé:** 14 of 15 Phase 1
  vulnerabilities are in code (only Vuln 12 is missing). Of the exposé's six
  pipeline areas, static analysis, fuzzing and SBOM/vulnerability management
  are done, artefact verification has started, while secure backend
  configuration checks and Docker/Kubernetes deployment are still open.

## Special note: `.gitleaksignore`

`.gitleaksignore` (repository root) tells Gitleaks which findings to skip. It
contains:

- **2 false positives** (the `$SONAR_TOKEN` lines), and
- **1 accepted real finding:** the leaked JWT key (Vuln 8), accepted only
  because the vulnerability is deliberate for this thesis. In a real project
  the key would have to be rotated (replaced) before accepting it.

Each entry is a fingerprint tied to one exact **commit ID**, file, rule and
line. Commit IDs change whenever the history is rewritten or copied into a
new repository (squashing, rebasing, history rewriting, starting a fresh repo
from the current files). After that, the entries no longer match anything.

**If the code is moved to a new repository (or its history is rewritten):**

1. **Remove the entries from `.gitleaksignore`** (or the whole file), since
   they would point at commit IDs that no longer exist.
2. **Run Gitleaks again** (`gitleaks git --redact --verbose`) and review what
   it reports in the new history. A fresh repository started from the current
   files would not contain the old `application.properties` at all, while a
   copied history would report it under new commit IDs.
3. **Only then re-add entries** for findings that are still false positives
   or still deliberately accepted, with their new fingerprints and a comment
   saying why.
4. Check that the pipeline fails on a test secret and passes without it, so
   the ignore file is not hiding anything by accident.

Never copy the old entries over unchecked: an ignore list that no longer
matches does nothing, and one that matches the wrong thing hides real leaks.

The same applies, in a weaker form, to the other suppressions added this week
(`nosemgrep` in `UserService.java`, `appClient/.mobsf`): each one names a
single rule and a reason, and should be re-checked when the code around it
changes.

## Challenges

- **Gitleaks reported a file that doesn't exist.** The reported path
  `src/main/resources/application.properties` (without `backend/`) only
  exists in old commits; the current `backend/.../application.properties`
  has no secret. Findings in a history scan always refer to the file as it
  was in the reported commit.
- **`mvn -q compile` looks like a hang in the dev container.** After a
  `pom.xml` change the dev loop prints `Change detected, recompiling...` and
  then nothing, because `-q` hides all output on success. Library changes in
  `pom.xml` also never reach the running app through hot reload; the backend
  has to be restarted (`docker compose restart backend`). The CI doesn't use
  hot reloading at all: it runs the built JAR and nginx with the built
  frontend.
- **Tools that only see part of the project.** Syft can't read
  `build.gradle.kts` (fixed with a lock file), and mobsfscan can't see SDK
  versions set in Gradle (false positive). Both had to be checked by hand
  before trusting their result.
- **Default settings that look safe but aren't.** Maven accepts a library
  with a wrong checksum without failing, and Syft returns an empty SBOM
  instead of an error when it finds no lock file. Both now fail explicitly.
- **Tool version compatibility.** `eslint-plugin-react` doesn't support
  ESLint 10, so the frontend uses ESLint 9.

## Still open

- **Phase 1:** Step 6 (webhook feature + Vuln 12, SSRF), and running and
  recording the verification checklist in `phase 1 spec.md` §11 (all 14 items
  still unticked).
- **Suppression policy:** decide per tool whether deliberate findings fail
  the pipeline or are documented and accepted, and apply it consistently. At
  the moment ESLint (Vuln 7) and Semgrep (Vuln 4, Vuln 6) fail, while
  Gitleaks, mobsfscan and Semgrep's Vuln 9 are suppressed. SonarCloud
  findings still need to be marked in SonarCloud itself.
- **Spec mismatches:** the `fuzz` job runs automatically (`when: always`),
  but the spec requires it to be manual; `/api/hello` exists although Step 0
  removed it.
- **CI clean-up:** `zap-ci.yml` and `zap-ci-test.yml` are now identical (the
  pipeline uses `zap-ci.yml`); delete one. Confirm in CI that
  `backend-grype` passes with Spring Boot 4.1.1.
- **App client:** `gradle-wrapper.jar` is ignored by the root `.gitignore`
  (`*.jar`), which blocks a future build job; no Android build/Lint job or
  SonarCloud project yet; the app ID is still `com.example.appclient`;
  `allowBackup="true"` needs a threat-model decision.
- **Exposé areas:** artefact verification is partial (no digest pinning of
  images, no hash-pinned `pip install`, no signing of our own SBOMs/images
  with Cosign, Syft/Grype downloads unverified); secure backend configuration
  checks (Hadolint, Checkov/KICS) and Docker/Kubernetes deployment are not
  started.
- **Docs:** `left-to-do.md` is outdated, `working_spec_finsec.md` is
  superseded by `phase 1 spec.md`, `CLAUDE.md` still describes CI as "just
  Semgrep".
- **Thesis writing** has not started yet (week 7 of 24).

## Where to start next week

Finish Phase 1 first: implement Step 6 (the webhook endpoint with Vuln 12 and
its frontend form), then go through the `phase 1 spec.md` §11 checklist once
and record the results, so Phase 1 is formally complete. In the same pass,
settle the suppression policy and the two spec mismatches (manual `fuzz` job,
`/api/hello`). After that, move on to the open exposé areas, starting with
Docker/Kubernetes deployment (image scanning and manifest checks), and begin
writing the tool-selection chapter in parallel from the findings collected in
the weekly progress files.
