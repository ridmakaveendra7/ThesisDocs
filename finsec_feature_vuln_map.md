# FinSec — feature vs. possible vulnerability

Feature selection driver for both phases: pick/keep the features most likely
to let a vulnerability slip in, not just the features needed for coverage of
the spec. See [`finsec_app_final_spec.md`](finsec_app_final_spec.md) for full
feature detail. Tools aren't limited to Semgrep/SonarQube/ZAP — several of
these are only realistically caught by a dedicated tool for that specific
bug class.

Rows are grouped by which layer actually carries the vulnerable code/config
— **Backend** (Spring Boot app logic), **Frontend** (the React SPA / browser
side), **Database** (schema, storage, credentials, and infra around the DB)
— since that's what determines who reviews it and which tool class actually
reaches it. SQL-injection-flavored rows are grouped under Database rather
than Backend: the defect is in application code, but the row is about how
the app talks to and constructs queries against the database, consistent
with how the frontend/database vulnerability-mapping work in
[`docs/Weekly Progress/week-04.md`](docs/Weekly%20Progress/week-04.md)
categorized them.

**Free tools only** — everything below is free/open source: Semgrep,
SonarQube (Community Edition), OWASP ZAP (including its add-ons), Jazzer,
Hydra, `ent` (entropy/randomness testing), and `race-the-web` (race-condition
testing).

## Phase 1 (base app, already built)

### Backend

| Feature | Possible vuln | Detection tool |
|---|---|---|
| User registration and login | The key used to sign login tokens is hardcoded in the code, and the registration confirmation shows the submitted username without cleaning it, so a script in it can run (XSS) | Semgrep, SonarQube, OWASP ZAP |
| Register a webhook URL for account notifications | The server fetches whatever URL it's given to "verify" it, so it can be tricked into calling an attacker's server (SSRF) | Semgrep, SonarQube |
| Built-in app monitoring/debug endpoints | Left open to anyone, exposing internal app details | OWASP ZAP, manual |
| Cross-origin request (CORS) settings | Any website is allowed to call the API using a logged-in user's credentials | Semgrep, SonarQube, OWASP ZAP |
| How errors are shown to the user | Full error details/stack traces are shown instead of a generic message | OWASP ZAP, manual |
| Cross-site request forgery (CSRF) protection | Protection is turned off | SonarQube |
| Logging login attempts (audit trail) | The submitted username is written into the log unescaped, so a crafted username can inject fake log lines (log injection) | Semgrep, SonarQube |
| Validating the session token on every request | A malformed or crafted token (bad encoding, wrong structure, unexpected claim values) isn't cleanly rejected, and either crashes the check or slips through it | Jazzer fuzz test |

### Frontend

| Feature | Possible vuln | Detection tool |
|---|---|---|
| Frontend rendering of the registration confirmation (React SPA) | The confirmation HTML from the server is inserted into the page as raw markup instead of plain text, so a crafted username's script actually runs in the browser — the backend's reflected-XSS bug chained into a real, DOM-based browser exploit | Manual review, OWASP ZAP (AJAX/JS-aware spider) |
| Where the login session token is kept in the browser | The token is stored in `localStorage`, which any script running on the page (e.g. via the XSS above) can read and steal | Manual review |
| Browser security headers (Content-Security-Policy) | No CSP is set, so a script injected via the XSS above runs with no extra browser-side restriction | OWASP ZAP (passive scan) |

### Database

| Feature | Possible vuln | Detection tool |
|---|---|---|
| Looking up a user by username during login | The login lookup builds its query by pasting the submitted username straight into the SQL text instead of using a safe parameterized query, so a crafted username can manipulate it (SQL injection) — this doesn't bypass the password check itself (passwords are hashed and compared in code, not in the query), but it can still be used to extract data via `UNION`-based injection or leak database structure through errors | Semgrep, SonarQube, OWASP ZAP |
| Database credentials configuration | A default/fallback database password committed to a tracked properties file (instead of only the gitignored `.env`) would be a hardcoded-credential leak, the same class as the JWT-secret finding above | Semgrep, SonarQube |
| Database container network exposure (docker-compose) | If the Postgres port were ever published to the host instead of kept on the internal Docker network only, it would be reachable directly, bypassing the app | Manual review |
| Database user privileges | The app's database role has full rights over its own database rather than least-privilege-scoped access | Manual review |

## Phase 2 (student-built)

Within each layer below, rows describing a feature from
[`temp_FinSec_Project_Specification_EN.md`](temp_FinSec_Project_Specification_EN.md)
come first (roughly in that document's own order — App client → functions
table → access matrix/data → device key/signature → banking API), then
rows with no counterpart there.

### Backend

| Feature | Possible vuln | Detection tool |
|---|---|---|
| App generates a key pair and registers the public key | The server doesn't properly check that a signature is genuine | Manual review, Jazzer fuzz test |
| Linking a bank account through the bank's own login page | The step that stops this flow being hijacked (the `state` value) isn't checked properly | Manual review, OWASP ZAP (manual request replay) |
| Where the user gets sent back to after bank login | The server sends the user (or itself) to an unchecked, attacker-controlled address | OWASP ZAP, Semgrep, SonarQube |
| The bank's own login/approval step (password + one-time code) | A flaw in how this step is handled lets someone skip proper approval | Manual review, OWASP ZAP (manual request replay) |
| Marking a session as "from the app" or "from the website" | A website-only session can pretend to be a full app session and get more access than it should | OWASP ZAP Access Control Testing add-on |
| Checking the app's digital signature on a payment | A fake, altered, or reused signature/payment gets accepted anyway | Manual review, Jazzer fuzz test |
| One-time code used to stop a request being replayed | The code can be reused or guessed | Semgrep, SonarQube, `ent` (entropy testing tool) |
| Checking payment amount, currency, and account number | Invalid values (e.g. negative or oversized amounts) get accepted | Semgrep, SonarQube, Jazzer fuzz test |
| Allowing small payments without extra approval, up to a limit | Sending several payments at once can slip past the limit check | `race-the-web` (open-source race-condition tool) |
| Showing a user their accounts, balances, and transactions | One user can view another user's data by changing an ID in the request | OWASP ZAP Access Control Testing add-on |
| Showing a user their calculated spending/income summary | One user can view another user's summary the same way | OWASP ZAP Access Control Testing add-on |
| Getting a new access token without logging in again | An old token can be reused, or a new session can be forced onto a user | Manual review, `ent` (entropy testing tool) |
| Keeping track of a user's logged-in session | An attacker can plant or reuse a session ID before the victim logs in | OWASP ZAP (Session Fixation rule) |
| Reading and using the data the bank sends back | Unexpected or malicious data from the bank isn't handled safely | Semgrep, SonarQube, Jazzer fuzz test |
| Letting a user view or delete their own stored data | One user can view or delete another user's data | OWASP ZAP Access Control Testing add-on |
| Logging in with a username and password | No limit on attempts, so passwords can be guessed automatically | Hydra |
| Audit logging of payment and consent events | Free-text bank data (e.g. creditor name, remittance info) is written into the log unescaped, so it can be used to inject fake log entries (log injection) | Semgrep, SonarQube |
| Downloading an account statement or payment receipt | The filename/report reference isn't checked, so a crafted name can read other files on the server (path traversal) — now pulling real statements/receipts built from live bank data, instead of Phase 1's placeholder files | Semgrep, SonarQube |

### Frontend

| Feature | Possible vuln | Detection tool |
|---|---|---|
| Displaying free-text fields from the bank (e.g. payment descriptions) | A hidden script in that text runs when it's displayed | OWASP ZAP, SonarQube |
| Client-side hiding of app-only features (derived metrics, large payments) | If the frontend only hides these behind a UI-level check instead of relying on the server's access-matrix enforcement, a developer could mistake the UI hiding for the real control, and a user could still reach them by calling the API directly | Manual review, OWASP ZAP Access Control Testing add-on |
| Frontend handling of the bank's SCA redirect callback | If the frontend also navigates using the redirect/state values without validating them itself, it duplicates the unchecked-redirect risk on the client side | Manual review, OWASP ZAP |
| Clickjacking protection on the payment-approval screen | The payment confirmation page could be framed in an invisible iframe to trick a user into approving a payment without realizing it | OWASP ZAP (passive scan for missing anti-clickjacking headers), manual review |
| Cross-origin messaging in the bank redirect flow | If `postMessage` or similar cross-window messaging is used without checking the sender's origin, another site could inject fake messages into the flow | Manual review |
| Caching account balances / derived metrics in the browser | If this financial data is stored client-side (e.g. for offline display) rather than only fetched live, it becomes readable by anything with access to the browser's storage/device | Manual review |

### Database

| Feature | Possible vuln | Detection tool |
|---|---|---|
| Storing the bank's access and refresh tokens | Tokens are stored in a way that could leak if the database were ever exposed | Manual review, SonarQube |
| Database queries for transaction/payment search built by students | The same SQL-injection mistake as Phase 1's login lookup could get repeated in new account/transaction/payment queries, now with real financial data at stake | Semgrep, SonarQube, OWASP ZAP |
| Storage of the three derived spending/income metrics | If this data isn't encrypted at rest, a database compromise exposes a full behavioural profile of the user — exactly the GDPR-sensitive data the metrics create in the first place | Manual review |
| Database backups / snapshots | If backup files containing bank tokens, transactions, or derived metrics aren't access-controlled the same way the live database is, they become an easier target | Manual review |
