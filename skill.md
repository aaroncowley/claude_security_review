# skill.md — Claude Code (Opus 4.5) Skill: Security Code Reviews

## Skill name
**security-code-review (Claude Code — Opus 4.5)**

## Purpose
Perform **repeatable, high-signal security code reviews** that:
- Find real vulnerabilities (not style nits)
- Explain exploitability clearly
- Provide minimal, correct fixes + tests
- Produce a review report that engineering teams can act on

## Scope (what this skill covers)
- Web apps, APIs, CLIs, backend services, IaC, CI/CD, auth flows
- Common vulnerability classes (OWASP Top 10, CWE)
- Secure design checks (threat modeling lite)
- Dependency / supply-chain risk triage (from lockfiles + manifests)
- Secrets and sensitive data handling

## Out of scope
- Full pentest / live exploitation
- “Prove exploit” PoCs that require running untrusted code in prod
- Malware development, offensive weaponization guidance

---

## Inputs
Provide one or more of:
- PR link or patch/diff
- Repo path(s) and key files
- Architecture notes (auth model, data flows)
- Known security requirements (SOC2, HIPAA, PCI, etc.)
- Threat model notes or “what worries you most”

Minimum viable input:
- A diff/PR + how it’s deployed (cloud, on-prem, public-facing, internal)

---

## Output artifacts
1. **Findings list** (prioritized)  
   Each finding includes: Title, Severity, Evidence, Impact, Exploit scenario, Fix, Tests, References (CWE/OWASP).
2. **Suggested patch snippets** (small + safe)  
3. **Verification checklist** for reviewers/QA  
4. **“No findings” note** (if applicable) with what was checked

---

## Severity taxonomy
Use this scale consistently:
- **Critical**: RCE, auth bypass, SQLi w/ data access, secrets exposure with blast radius, signing key compromise, arbitrary file write, SSRF to metadata, etc.
- **High**: privilege escalation, significant IDOR, sensitive data leakage, stored XSS, weak auth/session handling
- **Medium**: reflected XSS in limited context, CSRF in meaningful state change, insecure defaults, weak crypto usage without immediate break
- **Low**: hardening, missing security headers, verbose errors, minor DoS vectors
- **Info**: best practices, defensive coding suggestions

Also tag:
- **Exploitability**: Easy / Moderate / Hard
- **Impact**: Low / Medium / High
- **Confidence**: High / Medium / Low

---

## Review workflow (Opus 4.5)
### Phase 0 — Orientation (2–5 mins)
Goal: understand what changed and what it touches.
- What are the entry points? (HTTP handlers, RPC, message consumers, CLI args)
- What data is processed? (PII, tokens, payment data)
- What trust boundaries are crossed? (user → server, service → service, build → runtime)
- What changed in authN/authZ, validation, serialization, queries, templates?

### Phase 1 — Attack-surface mapping
Identify and list:
- Public endpoints and routes
- Serialization/deserialization boundaries
- File system operations
- DB query construction and ORM usage
- Template rendering / HTML contexts
- External calls (HTTP, DNS, shell, cloud metadata)
- Cryptographic operations and key management
- Authorization checks and object ownership logic

### Phase 2 — Deep review by vulnerability class
Use targeted passes:
- **AuthN/AuthZ**: session/token validation, role checks, object-level authorization (IDOR)
- **Injection**: SQL/NoSQL, command, template, LDAP, path traversal
- **XSS**: unescaped output, dangerous sinks, DOM injection (if JS)
- **SSRF**: URL fetch, webhook callbacks, image/proxy fetchers
- **Secrets**: env vars, logs, client exposure, test fixtures
- **Crypto**: homegrown crypto, weak modes, nonce reuse, JWT pitfalls
- **Deserialization**: unsafe loaders, polymorphic types, YAML/JSON pitfalls
- **Concurrency/race**: TOCTOU, double-spend style logic
- **DoS**: unbounded loops, regex backtracking, large payload parsing

### Phase 3 — Fixes + tests + guardrails
For each real issue:
- Provide a minimal code fix
- Add a regression test (unit/integration)
- Add a guardrail recommendation (lint rule, config, WAF rule, SAST pattern)

### Phase 4 — Final report
- Prioritize by **risk to production**
- Include “what I didn’t review” if inputs were partial

---

## Claude Code execution style (how to run the skill)
### System mindset
- Prefer **precision over volume**
- If uncertain, ask for the *missing artifact*, but still produce best-effort findings from what’s available
- Don’t invent endpoints, configs, or threat models—state assumptions clearly

### File reading strategy
- Start with the diff, then trace:
  - the call chain upward (entry point)
  - the data flow downward (sink)
- Open supporting files only when needed:
  - auth middleware, validators, serializers, ORM helpers, template utilities, HTTP clients, config

### “Stop doing” rules (reduce noise)
- Don’t spam generic OWASP lists
- Don’t flag “missing rate limiting” unless there’s a sensitive endpoint exposed
- Don’t mark every `TODO` as a vulnerability
- Don’t propose massive rewrites; patch minimally

---

## Required report format (copy/paste template)

### Summary
- **What changed:**  
- **Security posture impact:** (Improves / Neutral / Regresses)  
- **Top risks:** (1–3 bullets)

### Findings (prioritized)
For each finding:

**[Severity] Title**  
- **Location:** file:line (or function)  
- **Evidence:** code excerpt or description  
- **Impact:** what can happen  
- **Exploit scenario:** 2–5 steps  
- **Fix (minimal):** exact change recommended  
- **Tests:** what to add / update  
- **Tags:** CWE-XXX, OWASP category, Exploitability, Confidence

### Verification checklist
- [ ] AuthZ enforced at object level  
- [ ] Input validation at boundaries  
- [ ] Output encoding correct for context  
- [ ] No secrets in logs or responses  
- [ ] External fetches have allowlists + timeouts  
- [ ] Crypto uses vetted libs and safe defaults  
- [ ] Tests added for regressions

### Notes / assumptions / not reviewed
- …

---

## “Golden prompts” (use as Claude Code commands)
Use these as your starting messages.

### 1) Quick PR triage (high signal)
> Review this diff for security risks. Prioritize by exploitability and production impact. For each finding, include exact file locations, a minimal fix, and a regression test idea. If something is only a concern under assumptions, state them.

### 2) AuthZ / IDOR focused pass
> Trace authorization for all resource accesses changed in this diff. Look for IDOR/object ownership gaps, role bypasses, and missing server-side checks. Provide one concrete exploit scenario per issue.

### 3) Injection focused pass
> Identify any user-controlled data reaching SQL/NoSQL queries, shell commands, templates, file paths, or deserializers. Flag only issues with a plausible path. Provide safe coding alternatives.

### 4) SSRF / outbound fetcher pass
> Locate all outbound requests (HTTP/DNS). Check for SSRF, allowlist, redirect handling, timeout, DNS rebinding, and metadata endpoint access. Recommend hardened client settings.

### 5) Logging / secrets pass
> Find any place secrets/PII could be logged, returned, or stored insecurely (tokens, passwords, API keys). Suggest redaction patterns and safe logging conventions.

---

## Language-specific quick checks

### Node/TS
- `eval`, `Function`, `child_process`, unsafe template literals
- Express trust proxy / header spoofing
- `jsonwebtoken` alg confusion, missing `aud/iss`, weak expiry
- SSRF via `axios/fetch` + redirects

### Python
- `pickle`, `yaml.load` (unsafe), Jinja2 autoescape contexts
- `subprocess` with `shell=True`
- `requests` redirects + timeouts
- ORM raw queries and string formatting

### Java/Kotlin
- Deserialization (Jackson polymorphism), XXE, SSRF
- Spring Security method/route protections
- SQLi via string concatenation

### Go
- `html/template` vs `text/template`
- `exec.Command` arg handling
- `net/http` timeouts, redirect policies
- JWT validation footguns

### Ruby
- Rails strong params + mass assignment
- `ERB` contexts + `html_safe`
- YAML/pickle equivalents

---

## CI / guardrails recommendations
When a class of bug is found, propose one guardrail:
- SAST rule (Semgrep)
- Unit test pattern / fuzz test
- Dependency pinning / integrity
- Runtime hardening (CSP, headers, WAF, network egress)
- Secrets scanning in CI (pre-commit + pipeline)

Example guardrail snippet:
- “Add Semgrep rule to flag `yaml.load` without SafeLoader”
- “Add integration test that enforces object ownership on GET /resource/:id”

---

## Quality bar (what “done” means)
A review is complete when:
- All entry points touched by the change were traced to their sinks
- All High/Critical issues include:
  - exploit scenario
  - minimal patch
  - regression test idea
- Noise is low: findings are defensible and actionable
- Assumptions are stated explicitly

---

## Common failure modes (avoid these)
- Over-reporting “potential issues” without a data path
- Missing authorization bugs because you only looked at validators
- Confusing input validation with output encoding (XSS contexts!)
- Ignoring redirects/timeouts on outbound HTTP (SSRF/DoS)
- Recommending crypto rewrites instead of safe library usage

---

## Optional add-on: “Threat model lite” questions
Answer briefly before deep review:
1. Who is the attacker? (anon, logged-in user, insider service)
2. What assets matter? (PII, funds, tokens, infra)
3. What’s the worst-case? (data exfil, account takeover, RCE)
4. What changed that affects trust boundaries?

---

## Example “finding” (format reference)

**[High] IDOR: missing ownership check on document download**  
- **Location:** `api/documents.ts:88` in `getDocument(req, res)`  
- **Evidence:** uses `docId` from route and fetches document without verifying `doc.ownerId === req.user.id`  
- **Impact:** any authenticated user can download other users’ documents  
- **Exploit scenario:** login → guess/infer docId → GET `/documents/{id}` → receive чужой document  
- **Fix (minimal):** enforce ownership in query (`WHERE id=? AND owner_id=?`) and return 404  
- **Tests:** integration test ensuring user A cannot fetch user B’s document  
- **Tags:** CWE-639, OWASP A01 Broken Access Control, Exploitability: Easy, Confidence: High

---

## Operating constraints
- If code context is partial, label findings as:
  - **Confirmed** (direct path visible)
  - **Likely** (strong indicators but missing a file)
  - **Needs context** (requires config/infra detail)

End of skill.
