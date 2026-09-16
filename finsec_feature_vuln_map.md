# FinSec — feature vs. possible vulnerability

Feature selection driver for both phases: pick/keep the features most likely
to let a vulnerability slip in, not just the features needed for coverage of
the spec. See [`finsec_app_final_spec.md`](finsec_app_final_spec.md) for full
feature detail. Tools aren't limited to Semgrep/SonarQube/ZAP — several of
these are only realistically caught by a dedicated tool for that specific
bug class.

**Free tools only** — everything below is free/open source: Semgrep,
SonarQube (Community Edition), OWASP ZAP (including its add-ons), Jazzer,
Hydra, `ent` (entropy/randomness testing), and `race-the-web` (race-condition
testing).

## Phase 1 (base app, already built)

Same split as Phase 2: features the reference spec actually describes first,
then Phase-1-only carriers that exist purely to host a vulnerability and
have no counterpart in the reference spec.

### Features described in the reference spec

| Feature | Possible vuln | Detection tool |
|---|---|---|
| User registration and login | The key used to sign login tokens is hardcoded in the code, and the registration confirmation shows the submitted username without cleaning it, so a script in it can run (XSS) | Semgrep, SonarQube, OWASP ZAP |

### Features with no counterpart in the reference spec

| Feature | Possible vuln | Detection tool |
|---|---|---|
| Search for a user by name | The search text is pasted directly into a database query, so it can be used to manipulate the query (SQL injection) | Semgrep, SonarQube, OWASP ZAP |
| Download a report file by name | The filename isn't checked, so a crafted name can read other files on the server (path traversal) | Semgrep, SonarQube |
| Register a webhook URL for account notifications | The server fetches whatever URL it's given to "verify" it, so it can be tricked into calling an attacker's server (SSRF) | Semgrep, SonarQube |
| Built-in app monitoring/debug endpoints | Left open to anyone, exposing internal app details | OWASP ZAP, manual |
| Cross-origin request (CORS) settings | Any website is allowed to call the API using a logged-in user's credentials | Semgrep, SonarQube, OWASP ZAP |
| How errors are shown to the user | Full error details/stack traces are shown instead of a generic message | OWASP ZAP, manual |
| Cross-site request forgery (CSRF) protection | Protection is turned off | SonarQube |
| Logging login attempts (audit trail) | The submitted username is written into the log unescaped, so a crafted username can inject fake log lines (log injection) | Semgrep, SonarQube |
| Validating the session token on every request | A malformed or crafted token (bad encoding, wrong structure, unexpected claim values) isn't cleanly rejected, and either crashes the check or slips through it | Jazzer fuzz test |

## Phase 2 (student-built)

Rows are ordered to match
[`temp_FinSec_Project_Specification_EN.md`](temp_FinSec_Project_Specification_EN.md)'s
own structure: features it actually describes come first, in roughly the
order they appear there (App client → functions table → access matrix/data →
device key/signature → banking API); features with no counterpart in that
document are listed last, under their own heading.

### Features described in the reference spec

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
| Storing the bank's access and refresh tokens | Tokens are stored in a way that could leak if the database were ever exposed | Manual review, SonarQube |
| Displaying free-text fields from the bank (e.g. payment descriptions) | A hidden script in that text runs when it's displayed | OWASP ZAP, SonarQube |
| Getting a new access token without logging in again | An old token can be reused, or a new session can be forced onto a user | Manual review, `ent` (entropy testing tool) |
| Keeping track of a user's logged-in session | An attacker can plant or reuse a session ID before the victim logs in | OWASP ZAP (Session Fixation rule) |

### Features with no counterpart in the reference spec

| Feature | Possible vuln | Detection tool |
|---|---|---|
| Reading and using the data the bank sends back | Unexpected or malicious data from the bank isn't handled safely | Semgrep, SonarQube, Jazzer fuzz test |
| Letting a user view or delete their own stored data | One user can view or delete another user's data | OWASP ZAP Access Control Testing add-on |
| Logging in with a username and password | No limit on attempts, so passwords can be guessed automatically | Hydra |
| Audit logging of payment and consent events | Free-text bank data (e.g. creditor name, remittance info) is written into the log unescaped, so it can be used to inject fake log entries (log injection) | Semgrep, SonarQube |
