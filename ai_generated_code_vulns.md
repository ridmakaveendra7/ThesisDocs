# Common vulnerabilities in AI-generated code

## Most commonly introduced vulnerabilities

- **Cross-site scripting (XSS)** — the single most consistently failed check across studies; up to 86% of samples failed to defend against it, and 0 of 5 tested LLMs added any mitigating HTTP header. Detected by: Veracode's SAST platform; manual security checklist. [1][3]
- **Missing HTTP security headers** (CSP, X-Frame-Options, HSTS, Referrer-Policy) — none of 5 major LLMs (ChatGPT, Claude, Gemini, DeepSeek, Grok) implemented any of these when generating a full login/auth system. Detected by: manual checklist (NIST/OWASP-based). [3]
- **Log injection** — 88% failure rate in a large multi-model benchmark, the worst-performing category tested. Detected by: Veracode's SAST platform. [1]
- **Insufficiently random values** (weak randomness, CWE-330) — one of the top 3 most common weaknesses found in real Copilot-generated code pulled from live GitHub projects. Detected by: static analysis (unspecified tool). [2]
- **Improper control of code generation** (CWE-94) — also a top-3 weakness in that same real-world Copilot study. Detected by: static analysis (unspecified tool). [2]
- **No brute-force protection / no MFA / no CAPTCHA** — only 1 of 5 tested LLMs implemented account lockout; none implemented MFA or CAPTCHA. Detected by: manual checklist. [3]
- **Weak or missing CSRF protection** — only 1 of 5 tested LLMs implemented CSRF tokens. Detected by: manual checklist. [3]
- **Session management flaws** (missing Secure/HttpOnly/SameSite cookie flags, no session rotation/expiry) — inconsistent across models, several failed basic session-security checks. Detected by: manual checklist. [3]
- **SQL injection** — still a recurring OWASP Top 10 finding overall, though the best-handled class in most benchmarks (models use parameterized queries fairly reliably, unlike XSS/log injection). Detected by: Veracode's SAST platform; manual checklist. [1][3]
- **Exposed secrets/credentials in generated apps** — 400+ leaked secrets found scanning real "vibe-coded" applications. Detected by: automated vendor scanning (Escape.tech). [4]
- **Architectural/design-level flaws** (privilege escalation paths, broken auth design) — enterprise codebases using AI assistance showed a 322% rise in privilege-escalation paths and a 153% rise in broader design-level flaws. Detected by: not specified (vendor SCA/ASPM telemetry). [4]
- **Language matters**: in a large-scale scan of real AI-attributed GitHub code, 87.9% had no identifiable CWE-mapped vulnerability at all — but Python code showed markedly higher vulnerability rates (16–18%) than JavaScript (~9%) or TypeScript (2.5–7%). Detected by: CodeQL static analysis. [5]

## Developer trust is also a problem

Independent of vulnerability type: developers using AI coding assistants have been found to write *more* insecure code while reporting *higher* confidence in its security than a control group — and the models themselves often assign high confidence to code that is actually vulnerable. [6]

## Articles

1. Veracode — Spring 2026 GenAI Code Security Report: https://www.veracode.com/blog/spring-2026-genai-code-security/
2. Fu et al. — "Security Weaknesses of Copilot-Generated Code in GitHub Projects: An Empirical Study": https://arxiv.org/abs/2310.02059
3. Dora et al. — "The Hidden Risks of LLM-Generated Web Application Code": https://arxiv.org/pdf/2504.20612
4. Cloud Security Alliance — "Vibe Coding's Security Debt: The AI-Generated CVE Surge": https://labs.cloudsecurityalliance.org/research/csa-research-note-ai-generated-code-vulnerability-surge-2026/
5. Schreiber & Tippe — "Security Vulnerabilities in AI-Generated Code: A Large-Scale Analysis of Public GitHub Repositories": https://arxiv.org/pdf/2510.26103
6. "An Empirical Study of Security Calibration in Large Language Models for Code": https://arxiv.org/abs/2606.31159
