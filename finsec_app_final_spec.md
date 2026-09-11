# FinSec App — Final Specification (Phase 1 + Phase 2)

This is the authoritative specification for the FinSec application built in
this thesis. It's modeled on the structure of
[`temp_FinSec_Project_Specification_EN.md`](temp_FinSec_Project_Specification_EN.md)
(a practical-exercise brief from a related course) and adapts its system
design — PSD2 TPP concept, trust boundaries, two-level auth, access matrix,
device-key signature format, EuroTrust Bank sandbox API — into this project's
own two-phase build. Where this document adopts a design decision from that
reference material, it's stated here as this project's actual chosen design,
not just "inspiration" — this document is what supersedes that framing for
the parts it adopts.

**The build is split into two phases with different builders:**

| Phase | Built by | What it is |
|---|---|---|
| **Phase 1 — base app** | Course staff (already spec'd) | A minimal Spring Boot backend with just enough working auth to be useful, deliberately carrying a small set of vulnerabilities so the CI/CD pipeline (Semgrep, SonarQube, OWASP ZAP) has real findings from day one. Full build spec: [`working_spec_finsec.md`](working_spec_finsec.md). |
| **Phase 2 — the real FinSec app** | Students | The actual PSD2 TPP system: bank integration, device-key app auth, payments, the access matrix, derived metrics — built on top of Phase 1. Specified in full below (§5 onward). |

Phase 2 is not just additive: **remediating every vulnerability Phase 1
seeded is itself a required Phase 2 deliverable**, verified by a clean
pipeline run — see §5.4 and §6.7.

Cross-referenced documents (not repeated here in full):
- [`working_spec_finsec.md`](working_spec_finsec.md) — Phase 1's exact build spec (endpoints, vulnerability implementation detail, verification checklist).
- [`docs/Weekly Requirements/week-03.md`](docs/Weekly%20Requirements/week-03.md) and [`docs/Weekly Progress/week-03.md`](docs/Weekly%20Progress/week-03.md) — the vulnerability and fuzz-testing research this spec builds on.
- [`temp_FinSec_Project_Specification_EN.md`](temp_FinSec_Project_Specification_EN.md) — the original reference material.

---

## 1. Overview and purpose

FinSec is an account information and payment initiation service — a "Third
Party Provider" (TPP) in the sense of PSD2. On behalf of its users, FinSec
accesses their accounts at the **EuroTrust Bank** and initiates payments in
their name. FinSec is **not** the account-holding institution; the money
resides at the bank.

The exercise is about building this system *securely* across the full
software development lifecycle — requirements, design, implementation,
testing, integration, deployment, maintenance — with a strong emphasis on a
CI/CD pipeline that provides automated security evidence. This project adds
a second, deliberate layer on top of that general goal: students don't start
from a clean slate. They inherit a small, working, **already-vulnerable**
codebase (Phase 1) and must first use the pipeline to find and fix what's
wrong with it, before and while building the rest of the system (Phase 2) —
so the exercise covers both "build securely" and "recognize and remediate
insecurity you didn't write yourself," which is closer to real maintenance
work than a greenfield build would be.

The functionality itself is deliberately small. Three use cases carry the
whole system: linking a bank account, reading balances/transactions, and
initiating transfers.

---

## 2. System architecture

Three active parties, three trust boundaries. The trust boundaries are the
core of this exercise: each is a place where data crosses from a less
trusted into a more trusted zone and must be validated.

```
   [ App client ]         [ Web client ]
        |                      |
        |  Boundary A          |  Boundary B
        v                      v
   +-----------------------------------+
   |          FinSec backend           |
   +-----------------------------------+
                   |
                   |  Boundary C
                   v
           [ EuroTrust Bank ]
```

- **Boundary A** (App ↔ Backend): strongest channel. The app holds a device
  key and signs payment approvals. **Phase 2 work.**
- **Boundary B** (Web ↔ Backend): weaker channel. Password login only, no
  key. May read and initiate low-value payments. **Phase 2 work** (the web
  client doesn't exist in Phase 1).
- **Boundary C** (Backend ↔ Bank): FinSec is itself the client of a
  third-party service and must treat its responses with suspicion. **Phase 2
  work** (no bank integration exists in Phase 1).

Both clients share **one** FinSec backend and the same user and account
identity. **Phase 1 only builds a thin slice of that backend** — just enough
auth to exist and to carry the seeded vulnerabilities. None of the three
trust boundaries are meaningfully enforced yet in Phase 1: there's no app
client, no web client, and no bank integration, so Boundary A/B/C guarantees
described below don't hold until Phase 2 builds the pieces they depend on.
Phase 1's JWT-based login is a placeholder for "Level 2, FinSec's own auth"
(§6.1) — it has no channel concept (APP vs. WEB) yet; that distinction is
introduced in Phase 2.

---

## 3. Components

### 3.1 FinSec backend

Central component serving both clients, built across both phases.

**Phase 1 delivers:** username/password registration and login issuing a
JWT, plus the small set of carrier endpoints described in §5. No accounts,
transactions, payments, or bank integration.

**Phase 2 delivers**, on top of Phase 1:
- Communication with the EuroTrust Bank (consent flow, token management,
  account retrieval, payment initiation).
- Device public-key registration for the app channel, alongside Phase 1's
  existing password-based user model.
- Server-side custody of the bank tokens. **No** bank token, `client_secret`,
  or refresh token ever leaves the backend.
- Computation and storage of the three derived metrics (§6.4).
- Enforcement of the access matrix (§6.3) depending on channel.
- Remediation of every vulnerability Phase 1 seeded (§6.7).

Reference stack: Java 17+, Spring Boot, Maven, PostgreSQL. (The original
reference material specifies Java 21 and H2 in-memory; this project's actual
`pom.xml`/`docker-compose.yml` already target Java 17 and PostgreSQL
respectively — both satisfy Spring Boot's baseline, so there's no need to
force a change to match the reference material exactly.)

### 3.2 App client (key-capable) — Phase 2

A locally installed application on a **freely chosen target platform** (e.g.
Android, iOS, Windows desktop, Linux desktop). The choice of platform and its
key store is a design decision to be justified (§6.5). Responsibilities:

- Generate an asymmetric key pair; the private key stays on the device,
  protected by the platform-appropriate store.
- Perform first-time registration with FinSec, setting the web password in
  the process.
- Initiate payments of any amount, each with a signature over the concrete
  payment data.
- Retrieve the derived metrics (only this channel may).

For the reference/testing setup, the app is represented by a dependency-free
command-line signing tool (`FinsecCli`), not a full native app. The platform
itself is not graded — what's graded is whether the correct capabilities and
limits of the chosen platform are assessed (§6.5).

### 3.3 Web client — Phase 2

A browser-based front end on the same backend. Deliberately the weaker
channel:

- Password login (the password set during app registration).
- Read account list, balances, transactions.
- Initiate **low-value payments** up to the cumulative limit (§6.3), on the
  basis of the password session alone.
- **No** access to the derived metrics. **No** large payments.

Frontend framework/stack is unconstrained by this spec — pick whatever fits;
nothing in Phase 1 or this document assumes a particular frontend choice
(consistent with the backend-only scope of the vulnerability/fuzz research
this spec draws on).

---

## 4. What Phase 1 delivers (summary)

Full build detail lives in [`working_spec_finsec.md`](working_spec_finsec.md)
— this section only summarizes what Phase 2 inherits and is responsible for
dealing with. Phase 1 is **not** a partial implementation of Phase 2's
features to be extended in place; it's a small, separate, deliberately
imperfect starting point.

### 4.1 Endpoints handed to students

| Endpoint | Purpose | Carries |
|---|---|---|
| `POST /api/auth/register` | register user | hardcoded JWT secret, reflected XSS in the confirmation response |
| `POST /api/auth/login` | issue JWT | — |
| `GET /api/users/search?name=` | look up users by (partial) name | SQL injection |
| `GET /api/reports/download?file=` | download a report/statement file | path traversal |
| `POST /api/notifications/webhook-url` | register a webhook URL for account notifications | SSRF |
| `/actuator/**` | ops introspection | exposed/unauthenticated actuator |

Both `POST /api/auth/register` and `POST /api/notifications/webhook-url`
replace earlier carrier designs (`/api/greeting?name=` and
`/api/bank/redirect-check`) that turned out to be poor fits: a
smoke-test-only endpoint isn't a real FinSec feature, and a "bank redirect"
feature can't be justified in Phase 1 when EuroTrust integration is Phase 2
work. Both replacements carry the same vulnerability classes through
features Phase 1 can actually justify on its own.

Plus one working **Jazzer fuzz test** against the JWT/bearer-token
validation path, as a reference example (see `working_spec_finsec.md` §4).

### 4.2 Seeded findings students must triage

| Category / CWE | Detected by |
|---|---|
| Exposed, unauthenticated Actuator (CWE-200/16) | ZAP, manual |
| Permissive CORS (CWE-942) | Semgrep, SonarQube, ZAP |
| Reflected XSS (CWE-79) | ZAP, SonarQube |
| Verbose error responses (CWE-209/756) | ZAP, manual |
| Hardcoded JWT secret (CWE-798) | Semgrep, SonarQube |
| SQL injection (CWE-89) | Semgrep, SonarQube, ZAP |
| Path traversal (CWE-22) | Semgrep, SonarQube |
| SSRF (CWE-918) | Semgrep, SonarQube (ZAP needs extra setup) |
| CSRF disabled (CWE-352) | SonarQube hotspot |

Full detection rationale, exact code-level carriers, and the reasoning
behind each choice are in `docs/Weekly Progress/week-03.md`'s vulnerability
catalog — this table is only a pointer, not the source of truth.

### 4.3 What Phase 1 explicitly does not include

No accounts, transactions, payments, bank integration, device-key auth,
channel concept, access matrix, or derived metrics. All of §6 below is
Phase 2 scope, built from nothing on top of Phase 1's auth core.

### 4.4 Phase 2's obligation toward Phase 1's findings

Every row in §4.2 is a real, intentionally-planted finding — not a
hypothetical. Phase 2 must fix all of them as part of the exercise (not
optionally), with the pipeline's clean re-scan as the acceptance evidence.
This is a required, gradable Phase 2 deliverable, not incidental cleanup —
see §6.7.

---

## 5. Phase 2 — the FinSec application

Everything below is what students design and build, using Phase 1 as the
foundation.

### 5.1 Authentication — two separate levels

These two levels must not be conflated. Keeping them apart cleanly is a core
competency of this exercise.

**Level 1 — user towards the bank (PSD2 SCA).** Runs exclusively through the
redirect to the EuroTrust Bank with its TOTP procedure. FinSec is not
involved and never sees the bank credentials. This level is governed by the
PSD2 RTS.

**Level 2 — user towards FinSec.** The app authenticates via the device key,
the web client via the password. This level is **not** governed by the RTS,
because FinSec is not the account-holding institution; it is a risk-based
construction of FinSec's own. The device key is a **possession** factor;
only together with a device-side user verification (biometric/PIN) and the
dynamic linking to the payment data does the app approval reach the quality
of strong authentication.

Note: the device key is **not** a PSD2 SCA and is not required by PSD2. It's
FinSec's own protection mechanism. Phase 1's plain password/JWT login is a
Level-2 placeholder for the web-channel path only — the app-channel
device-key path is entirely new Phase 2 work, as is the channel marker that
distinguishes the two on every request.

### 5.2 FinSec functions

| Function | Channel | Notes |
|---|---|---|
| Register (create device key + password) | App | Proof of possession over a challenge |
| Link bank account | App (triggers bank SCA) | OAuth consent flow with `state` check |
| Log in | App (key) / Web (password) | Session carries channel marker |
| List accounts / balances / transactions | App + Web | Read-only |
| Initiate + approve payment (any amount) | App | Signature over payment data |
| Initiate + approve low-value payment | Web | Password session, up to cumulative limit |
| Read derived metrics | App only | App-bound session required |

Phase 1's register/login endpoints are the starting point for "Register" and
"Log in" above — they get extended with device-key support and a channel
marker, not thrown away.

### 5.3 Access matrix

| Action | App channel (key) | Web channel (password) |
|---|---|---|
| Login | key challenge | password |
| Read accounts/balances/transactions | yes | yes |
| Read derived metrics | yes | **no** |
| Payment of any amount | yes (signed) | **no** |
| Low-value payment (< €30) | yes (signed) | yes (password session) |

Low-value limits for the web channel (identical to the bank): single amount
below **€30**, sum since the last strong approval below **€100**, at most
**5** payments. The backend counts server-side; if a limit is exceeded, the
web-channel payment is rejected.

Reading the metrics requires an **app-bound session** — a password session
does not suffice, not even under the same user identity via the web channel.
The session carries a channel marker (APP or WEB) that the backend evaluates
on every request.

**Central enforcement rule:** all these checks happen server-side. No client
decides its own rights. In particular, a web session must not be able to
pose as an app session. (This is also one of the fuzz-testing candidates
noted in `docs/Weekly Progress/week-03.md` — the channel-marker
parsing/validation code is worth a dedicated fuzz test once it exists, since
a bug there undermines this entire rule.)

### 5.4 Data and the derived metrics

**What the backend stores:**
- User: username, password hash, associated public device key.
- Bank linkage: `consentId`, bank access token, refresh token — server-side
  only.
- Sessions with channel marker.
- The three derived metrics per user.

Raw data (balances, transactions) should be fetched live from the bank where
possible. Any storage of raw bank data beyond that is a decision to be
justified (purpose, retention, protection).

**The three metrics** FinSec computes from the transactions and stores:
1. Sum of expenses in the current month.
2. Sum of income in the current month.
3. Sum of transfers to a selectable recipient.

**Why this changes the legal situation:** once FinSec **derives and stores
new information** from bank data, it becomes a controller of a
**behavioural profile** under the GDPR — no longer a mere data intermediary.
A spending profile may, depending on content, even touch special categories
under Art. 9 (e.g. health-related or donation spending). Obligations to be
addressed in the threat model and data-protection concept: purpose
limitation (Art. 5(1)(b)), legal basis, data minimisation and retention, and
data-subject rights — access (Art. 15) and erasure (Art. 17) must cover the
**derived** metrics, not just the raw data.

### 5.5 Device key and its protection

The app generates a key pair (ECDSA over P-256). The private key stays on
the device. How it's protected is a design decision, assessed against five
requirements and located in the threat model:

1. Key material is non-exportable / does not leave process memory in
   cleartext.
2. Every use requires a user verification (biometric or PIN).
3. There is defined, justified behaviour if the chosen platform lacks the
   needed key-store function.
4. The residual risk of the chosen solution is named in the threat model.
5. For amounts above a self-chosen threshold, a justified stricter rule
   applies.

#### 5.5.1 Signature format (fixed — binding)

So that verification tooling works independently of the client platform,
this format is binding and must be implemented exactly:

- ECDSA over P-256 with SHA-256; signature as `r || s`, 32 bytes each,
  base64url without padding. Public key as JWK (`kty: EC`, `crv: P-256`).
- Canonical form with length prefixes (prevents ambiguity):

  ```
  F(x)          = <byte length of x as decimal> ":" <x>
  SIGNING_INPUT = "FINSEC-SIG-V1"
                || F(purpose)
                || F(nonce)
                || F(paymentId)
                || F(amountMinor)
                || F(currency)
                || F(creditorIban)
  ```

  `purpose` is `REGISTER`, `LOGIN`, or `PAYMENT`. For `REGISTER` and `LOGIN`
  the four trailing fields are empty (`0:`).
- Nonce: server-generated, ≥ 128 bit, valid 120 s, single-use, bound to
  purpose and (for `PAYMENT`) to user and `paymentId`.

A payment signature is always verified against the **server-side stored**
payment data, never against values from the request.

This exact format matters beyond just the app/backend contract: it's also
the primary target identified for unit-level, coverage-guided fuzz testing
(`docs/Weekly Progress/week-03.md`, "Fuzz-testing research") — a
hand-rolled, length-prefixed parser is exactly the kind of code where
mismatched lengths, embedded delimiters, and malformed empty fields cause
real bugs. **Building a Jazzer `@FuzzTest` against this parser is a required
part of implementing it**, not optional follow-up (§6.7).

### 5.6 Banking API integration (EuroTrust XS2A sandbox)

Base path `/xs2a/v1`. The bank is provided as a hardened Docker container and
is **not** modified by teams — it's the third party beyond Boundary C.
FinSec consumes these endpoints. Mandatory header `X-Request-ID` on every
request.

**Token** (`POST /oauth/token`, form-encoded) — three grants:
- `client_credentials` (+ `client_id`, `client_secret`, `scope=tpp`) →
  company token, 10 min, only usable to create a consent.
- `authorization_code` (+ `code`, `client_id`, `client_secret`,
  `redirect_uri`) → user token pair (`access_token` 30 min,
  `refresh_token`, `consent_id`).
- `refresh_token` (+ `client_id`, `client_secret`) → new token pair; the old
  refresh token is rotated out.

`client_id`/`client_secret` are sent **only** at this endpoint; all other
endpoints take only the token.

**Consent** (`POST /consents`, company token, header `TPP-Redirect-URI`) —
body includes `access`, `recurringIndicator`, `frequencyPerDay`. Returns
`consentId` and an `scaRedirect` link. The redirect URI must be registered
for the `client_id`. Implementing this correctly-validated redirect handling
is the real-world counterpart to the SSRF mistake Phase 1 seeds elsewhere
(§4.1's webhook-URL endpoint) — the same "don't fetch/redirect to an
unvalidated URL" discipline applies here too (§6.7).

**SCA redirect** — `GET /sca/authorize?consentId=...&state=...` (bank
login/consent form), `POST /sca/authorize` (approval, 302 back to
redirect). Approval requires username, password, and a **TOTP** code
(RFC 6238, six digits, 30-second window); each test user's TOTP secret ships
with the sandbox. The `state` parameter is mandatory and only reflected —
**FinSec must generate, bind, and verify it itself.** (Also a fuzz-testing
candidate per the research doc — this is exactly the kind of
self-implemented validation logic that's easy to get subtly wrong.)

**Account information** (`GET /accounts`, `.../balances`,
`.../transactions`; user token + `Consent-ID`) — `frequencyPerDay` is
counted server-side; exceeding it yields HTTP 429 with `Retry-After`.
`creditorName` and `remittanceInformationUnstructured` are free text from
beyond the trust boundary and **must be output-encoded before rendering** —
the same field also flagged in the fuzz research doc for parser-robustness
fuzzing distinct from the XSS-encoding concern.

**Payments** (`POST /payments/sepa-credit-transfers`, user token, header
`TPP-Redirect-URI`; `GET /payments/{paymentId}/status`) — initiation returns
`paymentId` and either an `scaRedirect` (SCA required) or, for a low-value
payment within the cumulative limits, directly status `ACCP` with
`"scaExempt": true` and no redirect. Status values: `RCVD`, `ACCP`, `RJCT`.
Transfers to an account at the same bank are credited to the recipient;
transfers to external IBANs leave the bank.

**Operations** — `POST /sandbox/reset?scope=all|payments|transactions`
(header `X-Sandbox-Token`), `GET /health`. `all` restores initial state,
`payments` clears payments/consents/tokens/counters, `transactions` empties
booking lists and resets balances. The `X-Sandbox-Token` is issued at
startup or set via `SANDBOX_TOKEN`.

**Error format:**
```
{ "tppMessages": [ { "category": "ERROR", "code": "<CODE>", "text": "..." } ] }
```
Codes: `FORMAT_ERROR` (400), `TOKEN_INVALID`/`TOKEN_EXPIRED`/`SCA_REQUIRED`
(401), `CONSENT_INVALID`/`CONSENT_UNKNOWN` (403), `RESOURCE_UNKNOWN` (404),
`ACCESS_EXCEEDED` (429), `INTERNAL_ERROR` (500).

Test users: `psu-001` … `psu-020`, passwords `Sandbox!001` … `Sandbox!020`,
each with at least one account; the first five have a second account. TOTP
secrets ship in `totp-secrets.txt`.

### 5.7 Security remediation and pipeline extension work

This is a required part of Phase 2, not a side task:

1. **Fix every Phase 1 finding** (§4.2), with the pipeline's clean re-scan
   (Semgrep + SonarQube + ZAP) as acceptance evidence, before or alongside
   feature work — a feature built next to an unfixed vulnerability doesn't
   satisfy this requirement.
2. **Wire whichever of SonarQube/ZAP aren't yet in `.gitlab-ci.yml`** — as of
   this spec, only Semgrep runs in the pipeline; adding SonarQube and ZAP
   (baseline and/or active scan, per the open question in
   `docs/Weekly Requirements/week-03.md`) is part of this work, not assumed
   to already exist.
3. **Extend fuzz-testing coverage as each new parsing/validation feature is
   built**, per the deferred list in `docs/Weekly Progress/week-03.md`'s
   fuzz research: the §5.5.1 signature-format parser, payment field
   validation, nonce handling, the `state` parameter, bank-response parsing,
   free-text bank data, and the channel marker. Phase 1 ships one working
   fuzz test (JWT validation) as the pattern to follow — each of these gets
   its own `@FuzzTest`, bounded with `maxDuration`/`maxExecutions`, added
   alongside the feature it tests, wired into the same on-demand
   (`when: manual`) CI job Phase 1 sets up.

---

## 6. Non-goals

- No requirement to build a production-grade bank — the EuroTrust sandbox is
  fixed, hardened, and not modified by teams.
- No mandated frontend framework or app platform — both are free choices,
  graded on justification rather than the specific choice made (§3.2, §3.3).
- No requirement to preserve Phase 1's code as-is where fixing a
  vulnerability requires rewriting it — §5.7's remediation work takes
  priority over keeping Phase 1's original implementation intact.

---

## 7. Document map

| Document | Covers |
|---|---|
| This document | Full Phase 1 summary + full Phase 2 spec |
| [`working_spec_finsec.md`](working_spec_finsec.md) | Phase 1's exact build spec: endpoints, per-vulnerability implementation detail, verification checklist |
| [`docs/Weekly Requirements/week-03.md`](docs/Weekly%20Requirements/week-03.md) | The research tasks that produced the vulnerability and fuzz-testing catalogs |
| [`docs/Weekly Progress/week-03.md`](docs/Weekly%20Progress/week-03.md) | The vulnerability catalog (detection rationale, CWEs, tool coverage) and fuzz-testing research (tooling, target mapping, CI approach) this spec's §4.2 and §5.7 summarize |
| [`temp_FinSec_Project_Specification_EN.md`](temp_FinSec_Project_Specification_EN.md) | Original reference material this document adapts its Phase 2 system design from |
