# Week 3 requirements

## Goal

Research which vulnerabilities to **intentionally introduce** into the FinSec
base implementation, so that the CI/CD pipeline's security tools have real,
positive findings to detect throughout the exercise. Target tools: **Semgrep**
(SAST), **SonarQube** (SAST / security hotspots), and — most importantly —
**OWASP ZAP** (DAST, i.e. it needs a *running* app and real HTTP traffic, not
just source code).

This week's deliverable is the **research and selection**, not the
implementation. Output goes in `docs/Weekly Progress/week-03.md` once done:
a vulnerability catalog (see shape below) and a short plan for the base
implementation delta. Actual code changes are a later week's work.

## Constraint driving the design: minimum base code, maximum vulnerabilities

Students are meant to build most of FinSec themselves (see
`temp_FinSec_Project_Specification_EN.md` for the kind of feature surface —
reference only, per [`CLAUDE.md`](../../CLAUDE.md)). We hand them a small base
implementation to extend, and that base is where the intentional
vulnerabilities live. So every candidate vulnerability should be judged on:

1. **Yield per line of base code** — prefer a one-line config change or a
   5-line endpoint over anything requiring a substantial feature to exist as a
   carrier. A single bad default (e.g. a permissive CORS bean, an exposed
   Actuator endpoint) is worth more than an elaborate vulnerable feature.
2. **Detectability by a *default* tool configuration** — assume Semgrep's
   default/registry rulesets, SonarQube's out-of-the-box Java security rules,
   and a ZAP baseline/active scan against the running app with no custom
   scripts, unless a candidate is explicitly flagged as "needs ZAP scripting"
   or "manual/logic-only" (see below). Don't credit a vulnerability as
   "detectable" on the strength of a rule that has to be hand-written for it.
   Cross-tool coverage matters more than what's technically dual-classifiable — 
   a vuln category all three tools happen to flag is worth more than three
   different single-tool findings, since it's what makes the pipeline's
   layered-detection story visible.
3. **Doesn't block the students' own build-out** — the vulnerable code should
   sit next to or underneath the features students add, not replace
   functionality they need to build themselves. The existing auth core
   (`AuthController`, `AuthService`, `JwtService`) is not off-limits — if a
   vulnerability genuinely belongs there (e.g. a flaw in how the JWT is
   issued/validated, or in password handling), introduce it there; just make
   sure students still have working register/login to build on top of.
4. **Thematically plausible for a banking TPP app** — favor vulnerability
   carriers that resemble real FinSec-shaped features (a redirect/callback
   URL, a transaction search, a file/report download, a device or session
   identifier) over generic throwaway demo endpoints, so the exercise still
   reads as a real app under review, not a vulnerability obstacle course.

## What to produce (catalog shape)

For each candidate vulnerability, capture:

- **Category / CWE** (e.g. SQL Injection / CWE-89).
- **Carrier** — the minimal code/config that introduces it, and roughly how
  many lines.
- **Detected by** — which of Semgrep / SonarQube / ZAP catch it by default,
  and *how* (static pattern match vs. security-hotspot rule vs. active/passive
  DAST scan). Note tool overlap explicitly.
- **Why it's realistic here** — the FinSec-shaped feature it hides behind.
- **Known caveats** — e.g. a secret that's only in a gitignored `.env` gives
  Semgrep/SonarQube nothing to scan; ZAP needs the app actually running in the
  pipeline with a reachable target and, for auth-gated endpoints, a valid
  session/token; a finding that only fires under an active (not passive/
  baseline) ZAP scan changes what the CI job needs to run.

## Candidate areas to investigate

Use this as a starting checklist for research, not a final decision — confirm
detectability against the actual tool configuration the pipeline will run
before committing a candidate to the catalog.

**Config/plumbing-level (near-zero code, good yield):**
- Actuator exposed and unauthenticated (`management.endpoints.web.exposure.include=*`
  reachable without auth) — information disclosure; strong ZAP + manual finding.
- Permissive CORS (wildcard origin combined with credentials allowed).
- Verbose error responses (stack traces / internal messages returned to the
  client) — information disclosure, mainly a ZAP/manual finding.
- CSRF disabled — already true in `SecurityConfig` (`csrf.disable()`); confirm
  whether SonarQube flags this as a security hotspot as-is, and decide whether
  to document it as an existing intentional finding rather than adding a new one.
- **Auth-core-level (now in scope — see constraint 3 above):**
  - Hardcoded/weak JWT signing secret committed directly in `SecurityConfig`
    or a scanned properties file, rather than only in the gitignored `.env`.
  - Username enumeration via distinguishable login error messages (invalid
    username vs. wrong password) — CWE-203; mainly a manual/ZAP finding.
  - No rate limiting / lockout on `/api/auth/login` — brute-force exposure;
    check whether ZAP's default scan actually flags this or if it's a
    manual/logic-only observation for this catalog.
- A hardcoded secret/credential actually committed to a scanned file (the
  current `jwt.secret` only lives in the gitignored `.env` — nothing for
  Semgrep/SonarQube to find there; a committed default/fallback value would be
  needed for this category to register at all).
- Missing security headers, to the extent they aren't already set by Spring
  Security's defaults — check what's actually missing before assuming this is
  free.

**Small carrier endpoints (5–15 lines each):**
- SQL Injection via a string-concatenated query on a small search-style
  endpoint (e.g. searching by name/description) instead of a parameterized one.
- Reflected XSS via an endpoint that echoes request input back into an
  `text/html` response unescaped.
- Path traversal via a file/report-download endpoint that builds a file path
  from unsanitized user input.
- SSRF via a server-side fetch of a user-supplied URL — plausible as an early
  stand-in for redirect-URI validation (the spec's `TPP-Redirect-URI` /
  `state` handling is exactly this kind of check done wrong).
- Weak/predictable randomness (`java.util.Random`/`Math.random()`) or a weak
  hash (MD5/SHA-1) used for a new token/code/reference value — note this is
  mainly a Semgrep/SonarQube finding, not something ZAP's default scan proves
  exploitable on its own.

**Logic-only flaws worth noting but likely tool blind spots (for later
discussion of pipeline limitations, not required for this week's tool-focused
catalog):**
- Mass assignment / privilege escalation (e.g. a role field accepted from the
  client instead of being server-assigned).
- Missing per-user authorization (IDOR) on an object-by-id endpoint — real DAST
  detection needs an authenticated multi-user ZAP scan setup, not the default
  baseline scan.

## Non-goals for this week

- No fixing of anything — this is about deciding what to introduce.
- No requirement to cover every category above; pick the set that best
  satisfies the yield/detectability/non-blocking/plausibility criteria.
- Don't design the CI job wiring for ZAP (needing the app up, auth for scanned
  routes, baseline vs. active scan choice) in detail yet — flag it as a known
  caveat per finding where relevant, but the pipeline job itself is later work.

## Open questions to resolve during research

- Does the current pipeline (`.gitlab-ci.yml` / `.gitlab/ci/static-analysis.yml`)
  even run SonarQube or ZAP yet, or only Semgrep? Confirm before assuming a
  candidate is "wired in" vs. "needs a new CI job."
- For ZAP: baseline (passive, spider-only) scan vs. full active scan changes
  which candidates above are actually reachable/detected — decide which mode
  the pipeline targets before finalizing the catalog's "detected by ZAP" claims.
