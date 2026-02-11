# Claude Security Code Review Skill (Opus 4.5)

A **repeatable, high-signal security code review** “skill” designed for use with **Claude Code (Opus 4.5)**. It focuses on finding **real vulnerabilities** (not style nits), explaining exploitability clearly, and producing **actionable fixes + tests** that engineering teams can ship.

## What this is

This repo contains a single skill definition: `skill.md`, which provides:

- A structured security review workflow (orientation → attack surface mapping → vuln-class passes → fixes/tests → report)
- A consistent severity taxonomy + tags (Exploitability / Impact / Confidence)
- A copy/paste **report template**
- “Golden prompts” you can run as Claude Code commands for different review modes (quick triage, AuthZ/IDOR, injection, SSRF, logging/secrets)
- Language-specific quick checks (Node/TS, Python, Java/Kotlin, Go, Ruby)
- Guardrail recommendations (Semgrep, tests, dependency pinning, runtime hardening, secrets scanning) 

## Scope

Covers: web apps, APIs, CLIs, backend services, IaC, CI/CD, auth flows, OWASP/CWE-class issues, threat-model-lite checks, dependency/supply-chain triage, and secrets/sensitive data handling.

Not covered: full pentests/live exploitation, PoCs requiring running untrusted code in production, and offensive weaponization guidance. 
## Inputs to provide (minimum + recommended)

**Minimum viable input**
- A PR/diff + how it’s deployed (cloud/on-prem, public-facing vs internal)

**Better inputs**
- PR link or patch/diff
- Repo paths + key files
- Architecture notes (auth model, data flows)
- Security requirements (SOC2, HIPAA, PCI, etc.)
- Threat model notes / “what worries you most”

## Outputs you should expect

1. **Prioritized findings list**: title, severity, evidence, impact, exploit scenario, minimal fix, tests, references (CWE/OWASP)
2. **Suggested patch snippets** (small + safe)
3. **Verification checklist** for reviewers/QA
4. **“No findings” note** (if applicable) with what was checked
## Severity taxonomy

- **Critical**: RCE, auth bypass, SQLi with data access, high-blast secrets exposure, signing key compromise, arbitrary file write, SSRF to metadata, etc.
- **High**: privilege escalation, significant IDOR, sensitive data leakage, stored XSS, weak auth/session handling
- **Medium**: reflected XSS (limited context), meaningful CSRF, insecure defaults, weak crypto usage without immediate break
- **Low**: hardening (headers), verbose errors, minor DoS vectors
- **Info**: best practices / defensive suggestions

Also tag:
- **Exploitability**: Easy / Moderate / Hard
- **Impact**: Low / Medium / High
- **Confidence**: High / Medium / Low 
## How to use

### Option A: Use the “Golden prompts” (recommended)

Copy/paste one of these into Claude Code and attach your diff/PR context.

**Quick PR triage**
> Review this diff for security risks. Prioritize by exploitability and production impact. For each finding, include exact file locations, a minimal fix, and a regression test idea. If something is only a concern under assumptions, state them. 
**AuthZ / IDOR pass**
> Trace authorization for all resource accesses changed in this diff. Look for IDOR/object ownership gaps, role bypasses, and missing server-side checks. Provide one concrete exploit scenario per issue.

**Injection pass**
> Identify any user-controlled data reaching SQL/NoSQL queries, shell commands, templates, file paths, or deserializers. Flag only issues with a plausible path. Provide safe coding alternatives.

**SSRF / outbound fetcher pass**
> Locate all outbound requests (HTTP/DNS). Check for SSRF, allowlist, redirect handling, timeout, DNS rebinding, and metadata endpoint access. Recommend hardened client settings.

**Logging / secrets pass**
> Find any place secrets/PII could be logged, returned, or stored insecurely (tokens, passwords, API keys). Suggest redaction patterns and safe logging conventions.

### Option B: Run the full workflow

Follow the phases in `skill.md`:

0. Orientation (entry points, data, trust boundaries, auth/validation/serialization/query/template changes)
1. Attack-surface mapping (routes, boundaries, FS ops, DB queries, templates, outbound calls, crypto, authZ)
2. Deep review by vulnerability class (AuthN/AuthZ, injection, XSS, SSRF, secrets, crypto, deserialization, race/TOCTOU, DoS)
3. Fixes + tests + guardrails
4. Final report (risk-to-prod, assumptions, what wasn’t reviewed) 
## Report template

A complete copy/paste report format is included in `skill.md` under **“Required report format”** and includes:
- Summary (what changed, posture impact, top risks)
- Findings (prioritized, with required fields)
- Verification checklist
- Notes / assumptions / not reviewed
## Guardrails

When you find a class of bug, the skill recommends proposing one guardrail such as:
- Semgrep/SAST rule
- Unit/integration test patterns (or fuzzing)
- Dependency pinning/integrity
- Runtime hardening (CSP/headers/WAF/egress restrictions)
- Secrets scanning (pre-commit + CI)
## Contributing

PRs welcome. If you add checks/workflows, keep the same philosophy:
- **Precision over volume**
- Minimal patches over rewrites
- Only flag issues with plausible data paths / exploitability
- Always include tests + verification steps for High/Critical findings
