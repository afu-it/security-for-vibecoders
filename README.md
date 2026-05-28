# security-for-vibecoders

> **Security for vibe coders.** Ship production code without getting sued. PDPA 2024 compliant. OWASP Top 10 assessed. Zero security knowledge required.

[![version](https://img.shields.io/badge/version-2.1-teal?style=flat-square)](./skills/pdpa-security-audit/SKILL.md)
[![works with](https://img.shields.io/badge/works%20with-Codex%20%7C%20Claude%20%7C%20Cursor%20%7C%20Windsurf-blue?style=flat-square)](#)
[![license](https://img.shields.io/badge/license-MIT-green?style=flat-square)](#)

---

## Why This Exists

A developer in Malaysia was charged in court for:
- SQL injection vulnerabilities (RM1,000,000 fine under PDPA 2024)
- Hardcoded secrets in public repos
- No audit logs (can't prove what happened)
- No testing before go-live
- GPL code with removed copyright notices

**PDPA 2024 has criminalized negligent Data Processors.** If you build a SaaS that handles user data and you ship insecure code, you face up to RM1,000,000 fine or 3 years prison.

This skill pack makes your AI agent enforce security automatically.

---

## Install

```bash
# Install into your current project
npx skills add afu-it/secure-ship

# Install globally (all projects)
npx skills add afu-it/secure-ship -g

# Preview before installing
npx skills add afu-it/secure-ship --list
```

Works with **Codex, Claude Code, OpenCode, Cursor, Windsurf**, and 40+ other agents.

---

## Skills Included

| Skill | What it does |
|-------|-------------|
| `pdpa-security-audit` | Full OWASP Top 10 + PDPA 2024 compliance audit of your codebase |
| `access-control-patterns` | Enterprise RBAC/ABAC, IDOR prevention, tenant isolation |
| `sql-injection-prevention` | Detect and fix SQL injection vulnerabilities in any framework |
| `input-validation` | Validate and sanitize all user input (XSS, path traversal, file uploads) |
| `secrets-management` | Prevent hardcoded secrets, .env setup, secret rotation |
| `security-headers` | Add CSP, HSTS, X-Frame-Options middleware to any framework |
| `auth-hardening` | Password hashing, JWT best practices, brute-force protection |
| `cors-configuration` | Safe CORS setup, prevent wildcard credential leaks |
| `data-encryption` | Encrypt PII at rest with AES-256-GCM, key management |
| `error-handling-security` | Sanitized error responses, no stack trace leaks |
| `audit-logging-workers` | Add structured audit logging with immutable R2 storage |
| `webhook-security` | Fix race conditions, add idempotency, timing-safe comparisons |
| `ci-security-gates` | Add CodeQL SAST, Gitleaks, dependency audit to CI/CD |
| `rate-limiting` | Per-user rate limiting to prevent abuse |
| `ssrf-prevention` | Enterprise SSRF defense, DNS rebinding protection, metadata blocking |
| `dependency-lockfile-audit` | Detect typosquatting, malicious packages, lockfile tampering |

---

## Quick Start

After installing, tell your AI agent:

```
Run a PDPA security audit on this codebase
```

Or use individual skills:

```
Check this codebase for SQL injection
Fix access control and tenant isolation issues
Validate all user input in my API routes
Fix hardcoded secrets in my project
Add security headers to my app
Harden my authentication system
Fix my CORS configuration
Encrypt PII fields in my database
Fix error messages leaking stack traces
Add audit logging to all my API routes
Fix the webhook race condition
Add security scanning to my CI pipeline
Add rate limiting to my API
Prevent SSRF in my URL preview feature
Audit my dependencies for malicious packages
```

---

## What Gets Checked (PDPA 2024 Forensic Findings)

Based on real Malaysian court cases where developers were charged:

| # | Forensic Finding | What the skill checks |
|---|-----------------|----------------------|
| 1 | Raw SQL with string concatenation | All DB queries use parameterized bindings |
| 2 | Path traversal in file downloads | No user-controlled file paths |
| 3 | No transaction safety / race conditions | Atomic operations, idempotency guards |
| 4 | Hardcoded secrets in repo | No API keys, passwords in source |
| 5 | No logging / audit trail | Structured audit logs on all sensitive ops |
| 6 | No testing before go-live | Tests + security audit in CI pipeline |
| 7 | Suspicious/unverified dependencies | Clean dependency tree, no typosquatting |
| 8 | GPL code with removed copyright | License compliance check |

---

## OWASP Top 10 Coverage

| Category | What the skill enforces |
|----------|------------------------|
| A01 Broken Access Control | Auth on all routes, RBAC/ABAC, tenant isolation, no IDOR |
| A02 Cryptographic Failures | HTTPS, timing-safe comparisons, no exposed secrets |
| A03 Injection | Parameterized queries, input validation |
| A04 Insecure Design | Rate limiting, idempotency, atomic transactions |
| A05 Security Misconfiguration | Security headers, no debug modes, sanitized errors |
| A06 Vulnerable Components | Dependency audit, lockfile verification |
| A07 Auth Failures | Session management, brute-force protection |
| A08 Data Integrity | SAST, secret scanning, webhook verification |
| A09 Logging & Monitoring | Structured audit logs, immutable storage |
| A10 SSRF | Safe fetch wrappers, DNS rebinding defense, private IP and metadata blocking |

---

## Legal Context (Malaysia)

| Law | Risk | What this prevents |
|-----|------|-------------------|
| PDPA 2010 (Pindaan 2024) Seksyen 5(1)(2) | RM1,000,000 / 3 tahun penjara | Kecuaian keselamatan data |
| Akta Hak Cipta 1987 Seksyen 41(1)(i) | RM2,000-RM20,000 per fail / 5 tahun | Pembuangan notis hak cipta |
| Akta Kontrak 1950 Seksyen 74 | Ganti rugi am + khas | Pecah kontrak kerana sistem tidak selamat |
| Kanun Keseksaan Seksyen 417 | 5 tahun penjara | Penipuan (menyembunyikan ketidakcekapan) |

---

## Supported Stacks

- **Node.js / TypeScript** (Express, Next.js, Hono, Fastify)
- **Cloudflare Workers** (D1, KV, R2, Durable Objects)
- **Python** (Django, Flask, FastAPI)
- **Laravel / PHP**
- **Any SQL database** (PostgreSQL, MySQL, SQLite, D1)

---

## Related Security Skill Libraries

This pack is intentionally focused on **PDPA 2024 Malaysia + OWASP Top 10 secure shipping** for developers building SaaS, web apps, and APIs.

If you need broader cybersecurity coverage beyond application go-live safety, check out:

| Project | Best for |
|---------|----------|
| [mukul975/Anthropic-Cybersecurity-Skills](https://github.com/mukul975/Anthropic-Cybersecurity-Skills) | Large cybersecurity skill library covering SOC, DFIR, threat hunting, malware analysis, cloud security, red team, pentesting, IAM, MITRE ATT&CK, NIST CSF, MITRE ATLAS, D3FEND, and NIST AI RMF |

Use `secure-ship` when you want a practical, code-level security pass before shipping production software. Use broader cybersecurity libraries when you need investigation, detection, incident response, or security operations playbooks.

---

## Contributing

Found a gap? Open an issue or PR at [github.com/afu-it/security-for-vibecoders](https://github.com/afu-it/security-for-vibecoders).

---

## License

MIT
