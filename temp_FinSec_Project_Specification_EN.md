# FinSec — Project Specification

Practical exercise "Secure Software Development", M.Sc. Applied Computer Science,
Fulda University of Applied Sciences.

Version 1.0 (English). This document specifies the FinSec system, the banking API
it consumes, and everything needed to build, run, and secure it. It is written
for a student tasked with setting up and testing the CI/CD security pipeline.

---

## 1. Overview and purpose

FinSec is an account information and payment initiation service — a "Third Party
Provider" (TPP) in the sense of PSD2. On behalf of its users, FinSec accesses
their accounts at the **EuroTrust Bank** and initiates payments in their name.
FinSec is **not** the account-holding institution; the money resides at the bank.

The exercise is about building this system *securely* across the full software
development lifecycle — identifying requirements, design, implementation,
testing, integration, deployment, maintenance — with a strong emphasis on a
CI/CD pipeline that provides automated security evidence.

The functionality itself is deliberately small. Three use cases carry the whole
system: linking a bank account, reading balances/transactions, and initiating
transfers.

---

## 2. System architecture

Three active parties, three trust boundaries. The trust boundaries are the core
of this exercise: each is a place where data crosses from a less trusted into a
more trusted zone and must be validated.

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

- **Boundary A** (App ↔ Backend): strongest channel. The app holds a device key
  and signs payment approvals.
- **Boundary B** (Web ↔ Backend): weaker channel. Password login only, no key.
  May read and initiate low-value payments.
- **Boundary C** (Backend ↔ Bank): FinSec is itself the client of a third-party
  service and must treat its responses with suspicion.

Both clients share **one** FinSec backend and the same user and account identity.

---

## 3. Components

### 3.1 FinSec backend

Central component serving both clients. Responsibilities:

- Communication with the EuroTrust Bank (consent flow, token management, account
  retrieval, payment initiation).
- User management: password (for the web channel) and public device key (for the
  app channel).
- Server-side custody of the bank tokens. **No** bank token, `client_secret`, or
  refresh token ever leaves the backend.
- Computation and storage of three derived metrics (Section 7).
- Enforcement of the access matrix (Section 6) depending on channel.

Reference stack: Java 21, Spring Boot, Maven, H2 in-memory. Runs on port 3000.

### 3.2 App client (key-capable)

A locally installed application on a **freely chosen target platform** (e.g.
Android, iOS, Windows desktop, Linux desktop). The choice of platform and its
key store is a design decision to be justified (Section 8). Responsibilities:

- Generate an asymmetric key pair; the private key stays on the device, protected
  by the platform-appropriate store.
- Perform the first-time registration with FinSec (Section 9), setting the web
  password in the process.
- Initiate payments of any amount, each with a signature over the concrete
  payment data.
- Retrieve the derived metrics (only this channel may).

For the reference/testing setup, the app is represented by a dependency-free
command-line signing tool (`FinsecCli`), not a full native app.

### 3.3 Web client

A browser-based front end on the same backend. Deliberately the weaker channel:

- Password login (the password set during app registration).
- Read account list, balances, transactions.
- Initiate **low-value payments** up to the cumulative limit (Section 6), on the
  basis of the password session alone.
- **No** access to the derived metrics. **No** large payments.

---

## 4. Authentication — two separate levels

These two levels must not be conflated. Keeping them apart cleanly is a core
competency.

**Level 1 — user towards the bank (PSD2 SCA).** Runs exclusively through the
redirect to the EuroTrust Bank with its TOTP procedure. FinSec is not involved
and never sees the bank credentials. This level is governed by the PSD2 RTS.

**Level 2 — user towards FinSec.** The app authenticates via the device key, the
web client via the password. This level is **not** governed by the RTS, because
FinSec is not the account-holding institution; it is a risk-based construction of
FinSec's own. The device key is a **possession** factor; only together with a
device-side user verification (biometric/PIN) and the dynamic linking to the
payment data does the app approval reach the quality of strong authentication.

Note: the device key is **not** a PSD2 SCA and is not required by PSD2. It is
FinSec's own protection mechanism.

---

## 5. FinSec functions

| Function | Channel | Notes |
|---|---|---|
| Register (create device key + password) | App | Proof of possession over a challenge |
| Link bank account | App (triggers bank SCA) | OAuth consent flow with `state` check |
| Log in | App (key) / Web (password) | Session carries channel marker |
| List accounts / balances / transactions | App + Web | Read-only |
| Initiate + approve payment (any amount) | App | Signature over payment data |
| Initiate + approve low-value payment | Web | Password session, up to cumulative limit |
| Read derived metrics | App only | App-bound session required |

---

## 6. Access matrix

| Action | App channel (key) | Web channel (password) |
|---|---|---|
| Login | key challenge | password |
| Read accounts/balances/transactions | yes | yes |
| Read derived metrics | yes | **no** |
| Payment of any amount | yes (signed) | **no** |
| Low-value payment (< €30) | yes (signed) | yes (password session) |

Low-value limits for the web channel (identical to the bank):
single amount below **€30**, sum since the last strong approval below **€100**,
at most **5** payments. The backend counts server-side; if a limit is exceeded,
the web-channel payment is rejected.

Reading the metrics requires an **app-bound session** — a password session does
not suffice, not even under the same user identity via the web channel. The
session carries a channel marker (APP or WEB) that the backend evaluates on every
request.

**Central enforcement rule:** all these checks happen server-side. No client
decides its own rights. In particular, a web session must not be able to pose as
an app session.

---

## 7. Data and the derived metrics

### 7.1 What the backend stores

- User: username, password hash, associated public device key.
- Bank linkage: `consentId`, bank access token, refresh token — server-side only.
- Sessions with channel marker.
- The three derived metrics per user.

Raw data (balances, transactions) should be fetched live from the bank where
possible. Any storage of raw bank data beyond that is a decision to be justified
(purpose, retention, protection).

### 7.2 The three metrics

FinSec computes from the transactions and stores:

1. Sum of expenses in the current month.
2. Sum of income in the current month.
3. Sum of transfers to a selectable recipient.

### 7.3 Why this changes the legal situation

Once FinSec **derives and stores new information** from bank data, it becomes a
controller of a **behavioural profile** under the GDPR — no longer a mere data
intermediary. A spending profile may, depending on content, even touch special
categories under Art. 9 (e.g. health-related or donation spending).

Obligations to be addressed in the threat model and data-protection concept:
purpose limitation (Art. 5(1)(b)), legal basis, data minimisation and retention,
and data-subject rights — access (Art. 15) and erasure (Art. 17) must cover the
**derived** metrics, not just the raw data.

---

## 8. Device key and its protection

The app generates a key pair (ECDSA over P-256). The private key stays on the
device. How it is protected is a design decision, assessed against five
requirements and located in the threat model:

1. Key material is non-exportable / does not leave process memory in cleartext.
2. Every use requires a user verification (biometric or PIN).
3. There is defined, justified behaviour if the chosen platform lacks the needed
   key-store function.
4. The residual risk of the chosen solution is named in the threat model.
5. For amounts above a self-chosen threshold, a justified stricter rule applies.

The platform itself is not graded — what is graded is whether the team correctly
assesses the capabilities and limits of its chosen platform.

### 8.1 Signature format (fixed)

So that the verification tooling works independently of the client platform, the
format is binding:

- ECDSA over P-256 with SHA-256; signature as `r || s`, 32 bytes each, base64url
  without padding. Public key as JWK (`kty: EC`, `crv: P-256`).
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

  `purpose` is `REGISTER`, `LOGIN`, or `PAYMENT`. For `REGISTER` and `LOGIN` the
  four trailing fields are empty (`0:`).
- Nonce: server-generated, ≥ 128 bit, valid 120 s, single-use, bound to purpose
  and (for `PAYMENT`) to user and `paymentId`.

A payment signature is always verified against the **server-side stored** payment
data, never against values from the request.

---
## 10. Banking API reference (EuroTrust XS2A sandbox)

Base path `/xs2a/v1`. The bank is provided as a hardened Docker container and is
**not** modified by teams — it is the third party beyond boundary C. FinSec
consumes these endpoints. Mandatory header `X-Request-ID` on every request.

### 10.1 Token

```
POST /oauth/token   (application/x-www-form-urlencoded)
```
Three grants:
- `client_credentials` (+ `client_id`, `client_secret`, `scope=tpp`) → company
  token, 10 min, only usable to create a consent.
- `authorization_code` (+ `code`, `client_id`, `client_secret`, `redirect_uri`)
  → user token pair (`access_token` 30 min, `refresh_token`, `consent_id`).
- `refresh_token` (+ `client_id`, `client_secret`) → new token pair; the old
  refresh token is rotated out.

`client_id`/`client_secret` are sent **only** at this endpoint; all other
endpoints take only the token.

### 10.2 Consent

```
POST /consents           (company token; header TPP-Redirect-URI)
```
Body includes `access`, `recurringIndicator`, `frequencyPerDay`. Returns
`consentId` and an `scaRedirect` link. The redirect URI must be registered for
the `client_id`.

### 10.3 SCA (redirect)

```
GET  /sca/authorize?consentId=...&state=...     → bank login/consent form
POST /sca/authorize                             → approval, 302 back to redirect
```
Approval requires username, password, and a **TOTP** code (RFC 6238, six digits,
30-second window). Each test user's TOTP secret is shipped with the sandbox and
can be imported into an authenticator app. The `state` parameter is mandatory and
is only reflected — FinSec must generate, bind, and verify it.

### 10.4 Account information (user token + `Consent-ID`)

```
GET /accounts
GET /accounts/{resourceId}/balances
GET /accounts/{resourceId}/transactions
```
`frequencyPerDay` is counted server-side; exceeding it yields HTTP 429 with
`Retry-After`. `creditorName` and `remittanceInformationUnstructured` are free
text from beyond the trust boundary and must be output-encoded before rendering.

### 10.5 Payments

```
POST /payments/sepa-credit-transfers   (user token; header TPP-Redirect-URI)
GET  /payments/{paymentId}/status
```
Initiation returns `paymentId` and either an `scaRedirect` (SCA required) or,
for a low-value payment within the cumulative limits, directly status `ACCP`
with `"scaExempt": true` and no redirect. Status values: `RCVD`, `ACCP`, `RJCT`.
Transfers to an account held at the same bank are credited to the recipient;
transfers to external IBANs leave the bank.

### 10.6 Operations

```
POST /sandbox/reset?scope=all|payments|transactions   (header X-Sandbox-Token)
GET  /health
```
`all` restores the initial state, `payments` clears payments/consents/tokens/
counters, `transactions` empties the booking lists and resets balances. The
`X-Sandbox-Token` is issued at startup or set via the `SANDBOX_TOKEN` env var.

### 10.7 Error format

```
{ "tppMessages": [ { "category": "ERROR", "code": "<CODE>", "text": "..." } ] }
```
Codes: `FORMAT_ERROR` (400), `TOKEN_INVALID`/`TOKEN_EXPIRED`/`SCA_REQUIRED`
(401), `CONSENT_INVALID`/`CONSENT_UNKNOWN` (403), `RESOURCE_UNKNOWN` (404),
`ACCESS_EXCEEDED` (429), `INTERNAL_ERROR` (500).

Test users: `psu-001` … `psu-020`, passwords `Sandbox!001` … `Sandbox!020`,
each with at least one account; the first five have a second account. TOTP
secrets are provided in `totp-secrets.txt`.

---

