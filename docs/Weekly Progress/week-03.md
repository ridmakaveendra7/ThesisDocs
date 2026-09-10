# Week 3 progress — vulnerability research

Research output for [`Weekly Requirements/week-03.md`](../Weekly%20Requirements/week-03.md).
Scope: **backend only** (Java/Spring Boot). The frontend stack isn't decided
yet, so anything inherently frontend-specific (DOM-based XSS, client-side
token storage choices, CSP tuned to a particular bundler/framework,
clickjacking mitigations that depend on how a chosen FE embeds pages) is
deliberately excluded — see the note at the end. Findings below are backend
config/code changes that a HTTP client of any kind (including `curl` or ZAP
itself) can trigger or that static analysis can see in the Java source.

Catalog shape follows the requirements doc: Category/CWE, Carrier, Detected
by, Why realistic, Known caveats.

## Tier 1 — config/plumbing-level (near-zero code)

### 1. Actuator exposed and unauthenticated
- **CWE**: CWE-200 (Information Exposure) / CWE-16 (Configuration).
- **Carrier**: add `spring-boot-starter-actuator`; set
  `management.endpoints.web.exposure.include=*` (or `env,heapdump,beanstacktrace,configprops`)
  and either omit an authorization rule for `/actuator/**` or explicitly
  `permitAll()` it in `SecurityConfig`. ~2 lines of config + 1 dependency.
- **Detected by**: ZAP (information disclosure alerts once it crawls
  `/actuator/env`, `/actuator/heapdump`, etc.); manual review; SonarQube less
  directly (it doesn't scan `application.properties` semantics for this by
  default, more a config-audit item). Strong, cheap ZAP finding.
- **Why realistic**: this is a real, currently-active CVE class —
  **CVE-2026-40976** is a critical (CVSS 9.1) Spring Boot 4.0.x actuator
  authorization bypass, and exposed `/actuator/env` + `/actuator/heapdump`
  together are "typically sufficient to escalate from anonymous access to full
  credential compromise" per public writeups (the Volkswagen heapdump incident
  is the canonical real-world case). It's exactly the failure mode operators
  hit in production.
- **Known caveats**: `finapp`'s `pom.xml` pins `spring-boot-starter-parent`
  4.1.0, which includes the fix for CVE-2026-40976 (forward-ported to the
  4.1.x line), and the app already defines its own `SecurityFilterChain`
  (the CVE only fires when there's *no* custom Spring Security config) — so
  this project isn't vulnerable to that CVE by accident. To get the finding
  we'd have to **deliberately** misconfigure the matcher (e.g. permit-all
  the actuator paths ourselves), which is the point: the vulnerability is an
  intentional authorization gap, not a reliance on the patched framework bug.

### 2. Permissive CORS (wildcard origin + credentials)
- **CWE**: CWE-942 (Overly Permissive CORS Policy).
- **Carrier**: a `CorsConfigurationSource` bean with `setAllowedOrigins(List.of("*"))`
  and `setAllowCredentials(true)` wired into `SecurityConfig`. ~5 lines.
  (Spring actually rejects `*` + credentials at startup in recent versions —
  use `setAllowedOriginPatterns(List.of("*"))` instead, which *is* accepted
  and equally permissive; worth using precisely because it's the realistic
  way this misconfiguration actually ships.)
- **Detected by**: ZAP passive scan (CORS misconfiguration alert); SonarQube
  ("CORS policies should be restrictive" — a well-known Sonar Java security
  rule, commonly cited as **S5122**; confirm the exact ID against the
  SonarQube version in use). Semgrep also has generic-CORS-wildcard rules in
  the `p/security-audit`/Spring rulesets.
- **Why realistic**: exactly the mistake teams make when a not-yet-decided
  frontend origin makes them "just allow everything for now" — thematically
  on point for this project's current state.
- **Known caveats**: only exploitable in a browser context with a client that
  sends credentialed cross-origin requests; until a frontend exists this is a
  static/tool finding rather than something with a live PoC. Still valid to
  seed now — the tools flag it from config alone.

### 3. Verbose error responses (stack traces / internal messages leaked)
- **CWE**: CWE-209 (Information Exposure Through an Error Message) / CWE-756.
- **Carrier**: `server.error.include-stacktrace=always`,
  `server.error.include-message=always`, `server.error.include-binding-errors=always`
  in `application.properties`. 3 lines, no code.
- **Detected by**: ZAP (information disclosure / application error alerts);
  manual review. Not typically a Semgrep/SonarQube source-scan finding since
  it's pure config, not a code pattern — flag it primarily as a ZAP/DAST item.
- **Why realistic**: an extremely common "we turned this on to debug and
  forgot" production mistake.
- **Known caveats**: needs an endpoint that actually throws (e.g. malformed
  JSON to `/api/auth/login`, or a `RuntimeException` like the one already
  thrown in `AuthService.register` for a duplicate username) to have
  something to trigger against during a scan.

### 4. Hardcoded/committed secret (make the JWT secret scannable)
- **CWE**: CWE-798 (Use of Hard-coded Credentials).
- **Carrier**: currently `jwt.secret` only resolves via `${JWT_SECRET}` from
  the gitignored `.env` — **nothing for Semgrep/SonarQube to find in the
  repo today**. To seed this finding, commit an actual default value, e.g.
  `@Value("${jwt.secret:Q7mK9xV2pL8sR4nT6wY3cF1hJ5zB0dG8uA2eM7qX9kP4vN6sW1rC3tH8yZ5fL0}")`
  in `SecurityConfig`, or put the literal value directly in
  `application.properties` (a committed file) instead of only in `.env`.
  1 line.
- **Detected by**: Semgrep (`java.lang.security.audit.*` hardcoded-secret
  rules and generic-secrets rulesets); SonarQube ("Hardcoded credentials
  should not be used" — commonly **S2068**, plus the newer S6437 credentials
  rule family the community forum discusses); both are core, high-confidence,
  default-enabled rules in each tool. Not a ZAP finding (ZAP tests the running
  HTTP surface, not source).
- **Why realistic**: hardcoded JWT signing keys are one of the most common
  real findings in Java Spring apps precisely because `@Value` defaults are
  an easy way to make local dev "just work."
- **Known caveats**: this is the one candidate that's currently *impossible*
  to detect with the existing repo layout — the requirements doc's "known
  caveats" point about `.env` applies literally here. Needs the value moved
  into a scanned file to register at all.

### 5. CSRF protection disabled
- **CWE**: CWE-352 (Cross-Site Request Forgery).
- **Carrier**: already present — `httpSecurity.csrf(csrf -> csrf.disable())`
  in `SecurityConfig.java`. Zero new code; this is an *existing* finding to
  document rather than introduce.
- **Detected by**: SonarQube ("Disabling CSRF protections is security-sensitive",
  commonly **S4502**) — a default security hotspot rule, very likely to fire
  on this exact line. Semgrep has equivalent `csrf-disabled`-style rules in
  Spring rulesets. Not directly a ZAP finding (ZAP doesn't distinguish "CSRF
  disabled" from "CSRF not needed" — it can flag missing anti-CSRF tokens on
  state-changing forms, but this API has none yet).
- **Why realistic**: CSRF-disable is boilerplate in almost every JWT-bearer-token
  Spring Security tutorial, copied without the reasoning that makes it safe.
- **Known caveats**: **whether this is actually exploitable depends entirely
  on the not-yet-decided frontend.** With a bearer-token-in-header client
  (current design) CSRF is genuinely not exploitable — the browser can't be
  tricked into attaching an `Authorization` header. It becomes a real,
  exploitable vulnerability only if a future frontend switches to
  cookie-based session transport. Keep this as a **static/hotspot finding
  now**, and flag it for re-evaluation once the frontend is chosen — this is
  a good example for the thesis of a tool finding whose real severity is
  context-dependent.

## Tier 2 — small carrier endpoints (5–15 lines each)

### 6. SQL injection via string-concatenated query
- **CWE**: CWE-89.
- **Carrier**: a small search-style endpoint (e.g. "search users by name" or,
  once a transaction-like entity exists, "search by description") built with
  `entityManager.createNativeQuery("SELECT * FROM users WHERE user_name LIKE '%" + q + "%'")`
  instead of a parameterized query/`@Query` with `:param`. ~8 lines including
  the controller method.
- **Detected by**: Semgrep (`java.lang.security.audit.sqli.*` and Spring-specific
  `java.spring.security.injection.tainted-sql-string` — a named registry rule
  that traces tainted `HttpServletRequest`/`@RequestParam` input into a SQL
  sink); SonarQube ("Database queries should not be vulnerable to injection
  attacks", commonly **S3649**); ZAP active scan (SQLi is one of its
  strongest, best-supported active-scan rule families — error-based and
  boolean-based detection both apply here). This is the strongest
  three-way-overlap candidate in the whole catalog.
- **Why realistic**: a "search transactions/accounts" endpoint is exactly the
  kind of feature the FinSec-shaped app will plausibly grow, and search-by-text
  is the single most common real-world carrier for SQLi.
- **Known caveats**: needs the endpoint reachable without auth (or ZAP needs a
  valid token) for the active scan to reach it — decide auth context for the
  ZAP job.

### 7. Reflected XSS (backend fails to output-encode)
- **CWE**: CWE-79.
- **Carrier**: an endpoint that echoes a request parameter back inside an
  `text/html`-typed response body unescaped — e.g. extend the existing
  `hello.java` controller's `/api/hello` to accept `?name=` and return
  `"Hello, " + name` with `produces = MediaType.TEXT_HTML_VALUE`. ~4 lines.
  A JSON-only API returning `application/json` is *not* an XSS carrier (the
  browser won't execute it), so the response content type matters here.
- **Detected by**: ZAP active scan (reflected-XSS is one of its best-supported,
  highest-confidence default active rules); SonarQube has a reflected-XSS
  rule for Java web endpoints (commonly **S5131**, "Endpoints should not be
  vulnerable to reflected Cross-Site Scripting attacks") when it can trace an
  `@RequestParam` into an HTML-producing response; Semgrep has taint-mode
  Spring XSS rules too, though coverage is generally weaker than for SQLi.
- **Why realistic**: the spec's own bank-data notes (free-text
  `creditorName`/remittance fields "must be output-encoded before rendering")
  point at exactly this class of bug — a support/preview endpoint echoing
  free text is a very plausible stand-in in the meantime.
- **Known caveats**: purely backend-triggerable and framework-agnostic (it's
  about the Java controller not encoding output, not about any frontend
  templating choice) — safe to include despite the frontend being undecided.

### 8. Path traversal on a file/report endpoint
- **CWE**: CWE-22.
- **Carrier**: a "download report/receipt" endpoint that builds a file path
  as `new File(baseDir, filename)` (or `Paths.get(baseDir, filename)`) from a
  request parameter with no normalization/allowlist check, then serves it.
  ~10 lines.
- **Detected by**: Semgrep has a named Spring registry rule for exactly this
  (`java.spring.security.injection.tainted-file-path`, tracing tainted
  request input into a file-path sink); SonarQube ("I/O function calls should
  not be vulnerable to path injection attacks", commonly **S2083**); ZAP's
  active scan includes path-traversal payloads (`../../../etc/passwd` style)
  against parameters that look file-path-shaped, though its hit rate here is
  lower than for SQLi/XSS since it depends on ZAP guessing the parameter is
  file-related.
- **Why realistic**: a consent/statement/receipt PDF download is a natural
  future FinSec feature; "map by filename" instead of "map by opaque ID"
  is the textbook real-world mistake.
- **Known caveats**: ZAP detection is the least reliable of the three tools
  here — treat SonarQube/Semgrep as the primary detectors for this one and
  ZAP as a bonus, not the headline tool for this candidate.

### 9. SSRF via unvalidated server-side fetch of a client-supplied URL
- **CWE**: CWE-918.
- **Carrier**: an endpoint that does a server-side HTTP call (`RestTemplate`/
  `HttpClient`) to a URL taken from the request body/params with no
  allowlist — framed as an early stand-in for validating a redirect/callback
  URL (the spec's `TPP-Redirect-URI` concept is exactly this kind of check,
  done wrong). ~10 lines.
- **Detected by**: Semgrep (`java.spring.security.injection.tainted-url-host`
  is a named registry rule tracing tainted input into an outbound HTTP host);
  SonarQube has SSRF-adjacent rules for unvalidated URL construction in HTTP
  clients; ZAP can detect SSRF if configured with an out-of-band collaborator
  or by pointing the target at a ZAP-controlled listener — this needs a
  slightly more deliberate DAST setup than the baseline scan, so don't assume
  it's "free" the way SQLi/XSS are.
- **Why realistic**: directly foreshadows real bank-integration code
  (redirect-URI / `state` handling) that the students will eventually have to
  get right for real.
- **Known caveats**: full ZAP exploitability confirmation needs either an
  active-scan add-on or an OOB collaborator reachable from the CI runner —
  note this as a pipeline-wiring dependency, not just an app-code decision.

### 10. XXE via a permissive XML parser (if/when any XML input is accepted)
- **CWE**: CWE-611.
- **Carrier**: only relevant if a feature ever parses client-supplied XML
  (e.g. a "bulk import" or SEPA-XML-style feature). `DocumentBuilderFactory`
  is **XXE-vulnerable by default** in Java unless external entities are
  explicitly disabled — so simply *not adding* the standard hardening
  (`setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)`,
  disabling `external-general-entities`/`external-parameter-entities`) is
  enough; no "extra" vulnerable code is needed, only the absence of 2–3 lines
  of hardening. ~0 lines added beyond whatever XML-parsing feature exists.
- **Detected by**: Semgrep (`java.lang.security.audit.xxe.*`, a very standard,
  high-confidence default rule for un-hardened `DocumentBuilderFactory`);
  SonarQube ("XML parsers should not be vulnerable to XXE attacks", commonly
  **S2755**); ZAP active scan includes XXE payloads but only reaches them if
  it can identify an XML-accepting parameter/content-type, so this is again
  primarily a SAST-tool finding.
- **Why realistic/caveat**: **hold this one until an actual XML-accepting
  feature exists** — bank/PSD2 APIs are JSON per the spec, so don't invent an
  XML endpoint purely to carry this vulnerability; it fails the "thematically
  plausible" criterion otherwise. Listed here for completeness/future weeks.

### 11. Weak/predictable randomness or hash for a new token/code value
- **CWE**: CWE-330 (weak randomness) / CWE-327 (weak hash, e.g. MD5/SHA-1).
- **Carrier**: use `java.util.Random` or `Math.random()` (instead of
  `SecureRandom`) to generate any new security-relevant value (a reset code,
  a device-pairing code, an idempotency key), or hash such a value with MD5.
  ~2 lines wherever such a value is first introduced.
- **Detected by**: Semgrep (`java.lang.security.audit.crypto.*` — predictable
  random / weak hash rules are core, default-enabled); SonarQube ("Using
  pseudorandom number generators (PRNGs) is security-sensitive", commonly
  **S2245**; "Using weak hashing algorithms is security-sensitive", commonly
  **S4790**) — both are standard default hotspot rules. Not a ZAP finding —
  weak randomness isn't observable from HTTP responses alone without a
  statistical/predictability analysis ZAP doesn't do by default.
- **Why realistic**: exactly the kind of thing developers reach for
  reflexively for "just a code, not a password."
- **Known caveats**: needs a carrier feature that generates such a value to
  exist yet — don't invent a throwaway "random code" endpoint purely to host
  this; wait until a real one (e.g. a pairing/reset code) is being built.

## Tier 3 — auth-core-level (now in scope per the requirements doc)

### 12. Hardcoded JWT secret directly in `SecurityConfig`
Same underlying issue as candidate 4, called out separately because it lives
in the auth core specifically. `SecurityConfig.jwtSecretKey` currently reads
`@Value("${jwt.secret}")` with no default — adding a fallback default value
(`"${jwt.secret:<literal-secret>}"`) is the minimal one-line change and was
already covered above; don't double-count it in the final selected set.

### 13. Username enumeration via distinguishable login errors
- **CWE**: CWE-203.
- **Carrier**: none needed to *introduce* — verify whether Spring Security's
  default `AuthenticationManager`/`UserDetailsService` failure path already
  distinguishes "user not found" (thrown explicitly in `UserService.
  loadUserByUsername` as `UsernameNotFoundException`) from "bad password"
  in status code, response body, or timing. If Spring's default
  `DaoAuthenticationProvider` normalizes both to the same generic
  `BadCredentialsException`/401 (which it does by default), this is *not*
  currently exploitable and would need deliberate weakening (e.g. catching
  and rethrowing `UsernameNotFoundException` with a distinct message) to
  become one.
- **Detected by**: mainly manual/DAST observation (ZAP doesn't flag this by
  default without a differential-response active-scan rule configured) —
  treat as a manual-testing / logic-review item, not a default-tool finding.
- **Known caveats**: low priority for this catalog given constraint 2
  (default-tool detectability) — keep on the list for completeness/manual
  test coverage discussion, not as a headline Semgrep/SonarQube/ZAP finding.

### 14. No rate limiting / lockout on `/api/auth/login`
- **CWE**: CWE-307.
- **Carrier**: none needed — the endpoint has no throttling today.
- **Detected by**: not a default ZAP baseline/active-scan finding (ZAP has no
  brute-force alert out of the box; its "Fuzzer" can be pointed at the
  endpoint manually but that's not part of an automated pipeline run).
  SonarQube/Semgrep don't reason about missing rate-limiting either. This is
  a genuine **tool blind spot** — useful in the thesis as a contrast case
  (a real, well-known OWASP risk category that the three chosen tools, run
  in their default pipeline configuration, will not surface), not a catalog
  entry to count on for a "detected" finding.

## Tool-detection quick reference (from this research)

| Tool | What it actually sees | Strong here | Weak/blind here |
|---|---|---|---|
| **Semgrep** | Java source, pattern + light taint tracking | Hardcoded secrets, SQLi/XSS/SSRF/path-traversal taint rules (several with **named** Spring registry rule IDs), XXE, weak crypto | Anything config-only (Actuator exposure, verbose errors) that isn't a source-code pattern; business-logic flaws (mass assignment, missing per-object authorization) |
| **SonarQube** | Java source + design-level "security hotspots" | Same injection classes as Semgrep plus dedicated hotspot rules for CSRF-disabled, CORS wildcard, weak randomness/hashing, XXE | Also blind to pure-config/runtime issues and to logic flaws; hotspots require a human to *review and mark reviewed*, so an unreviewed hotspot may not "fail" a pipeline the way a Semgrep/SAST error gate does — confirm how the pipeline gates on Sonar hotspots vs. issues |
| **OWASP ZAP** | Live HTTP traffic against the *running* app | SQLi (best-supported active-scan family), reflected XSS, information disclosure (verbose errors, exposed Actuator data), missing/misconfigured security headers, CORS misconfig | Source-only issues (hardcoded secrets, weak crypto choice), anything needing multi-user/authenticated context (IDOR), brute-force/rate-limiting, anything requiring OOB confirmation (deep SSRF) without extra scan configuration |

## Explicitly excluded (frontend-specific, per this week's scope)

Left out because they depend on a frontend framework/transport choice that
doesn't exist yet — revisit once the frontend is decided:
- DOM-based XSS, client-side template injection.
- Where the auth token is stored client-side (`localStorage` vs. cookie) and
  the token-theft/XSS implications of that choice.
- Content-Security-Policy tuned to a specific bundler/framework's inline
  script/style needs.
- Clickjacking mitigation specifics that depend on whether/how the frontend
  embeds pages in iframes.
- `SameSite`/cookie-flag choices tied to a same-origin vs. cross-origin
  frontend deployment.

(CSRF-disable and CORS-wildcard are *kept* in the catalog above despite
depending partly on the frontend, because the misconfiguration itself is
100% backend code/config and both Semgrep/SonarQube flag it from the backend
source alone — only their real-world *exploitability* is frontend-dependent,
which is called out in each entry's caveats.)

## Sources

- [CVE-2026-40976: Spring Boot 4.0 Actuator Authorization Bypass — HeroDevs](https://www.herodevs.com/blog-posts/cve-2026-40976-spring-boot-4-0-actuator-authorization-bypass)
- [CVE-2026-40976 — spring.io official advisory](https://spring.io/security/cve-2026-40976/)
- [CVE-2026-40976 — ZeroPath Blog](https://zeropath.com/blog/cve-2026-40976-spring-boot-actuator-authorization-bypass)
- [Exploring Spring Boot Actuator Misconfigurations — Wiz Blog](https://www.wiz.io/blog/spring-boot-actuator-misconfigurations)
- [Spring Boot Actuator Security: Endpoints, Risks & CVEs — Cyberphinix](https://cyberphinix.de/en/blog/spring-boot-actuator-explained/)
- [Semgrep Registry — java ruleset](https://registry.semgrep.dev/ruleset/java)
- [Semgrep java rules — Boost Security](https://docs.boostsecurity.io/rules/semgrep_java.html)
- [Path Traversal from HTTP Request Data in File Access in Spring — Sourcery vulnerability DB](https://www.sourcery.ai/vulnerabilities/java-spring-security-injection-tainted-file-path)
- [SonarQube Server — Security-related rules](https://docs.sonarsource.com/sonarqube-server/10.8/user-guide/rules/security-related-rules)
- [Hardcoded Credentials (S6437, S2053, S2068) — Sonar Community](https://community.sonarsource.com/t/hardcoded-credentials-s6437-s2053-s2068/89853)
- [OWASP ZAP – Missing Anti-clickjacking Header alert (10020)](https://www.zaproxy.org/docs/alerts/10020-1/)
- [ZAP Passive Scan Rules](https://www.zaproxy.org/docs/desktop/addons/passive-scan-rules/)
- [Active Scan — Zed Attack Proxy (ZAP)](https://www.zaproxy.org/docs/desktop/start/features/ascan/)
- [How to Implement API Security Testing with OWASP ZAP](https://oneuptime.com/blog/post/2026-01-25-owasp-zap-api-security/view)
- [Prevent XML External Entity Vulnerabilities for Java — Semgrep cheat sheet](https://semgrep.dev/docs/cheat-sheets/java-xxe)
- [XML External Entity Prevention — OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html)
- [Insecure Deserialization: Finding Java Vulnerabilities with CodeQL — GitHub Security Lab](https://securitylab.github.com/research/insecure-deserialization/)
- [Deserialization Vulnerabilities in Java — Baeldung](https://www.baeldung.com/java-deserialization-vulnerabilities)
- [Spring Path Traversal Guide: Examples and Prevention — StackHawk](https://www.stackhawk.com/blog/spring-path-traversal-guide-examples-and-prevention/)
- [Regular expression Denial of Service - ReDoS — OWASP](https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS)
- [JWT Algorithm Confusion Attack (RS256 vs HS256) — Sourcery vulnerability DB](https://www.sourcery.ai/vulnerabilities/jwt-algorithm-confusion)
- [Algorithm confusion attacks — PortSwigger Web Security Academy](https://portswigger.net/web-security/jwt/algorithm-confusion)
- [OWASP WebGoat — GitHub](https://github.com/WebGoat/WebGoat)
- [OWASP WebGoat | OWASP Foundation](https://owasp.org/www-project-webgoat/)
- [Damn Vulnerable Java (EE) Application — appsecco/dvja](https://github.com/appsecco/dvja)
- [Top 25 Most Dangerous Software Weaknesses of 2025 — Infosecurity Magazine](https://www.infosecurity-magazine.com/news/top-25-dangerous-software/)
- [OWASP Top 10 2026: All 10 Risks Explained — Reflectiz](https://www.reflectiz.com/blog/owasp-top-ten-2026/)
