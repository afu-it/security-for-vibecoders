---
name: pdpa-security-audit
description: "Full PDPA 2024 + OWASP Top 10 security audit for any codebase. Checks all 8 forensic findings from Malaysian court cases plus enterprise controls for access control, SSRF, secrets, encryption, CORS, auth hardening, and error handling. Produces a compliance scorecard and actionable fix list. Use when asked to audit security, check PDPA compliance, run OWASP assessment, or before go-live."
version: "1.1"
---

# PDPA Security Audit

Run a comprehensive security audit mapped to PDPA 2024 (Malaysia) and OWASP Top 10 (2021). This skill is designed for developers who may not have formal security training but need to ship compliant production code.

## When to Use

- Before go-live / production deploy
- When asked "is my code secure?"
- When asked about PDPA compliance
- When asked to run a security audit
- When asked about OWASP
- After major feature additions that touch auth, payments, or user data

## Audit Process

### Phase 1: Forensic Findings Check (PDPA 2024)

Check the codebase against all 8 findings from real Malaysian court cases:

#### Finding 1: SQL Injection

Search for:
- `DB::raw()`, `whereRaw()`, `orderByRaw()` with string concatenation (Laravel)
- Template literals or string concatenation in SQL queries (Node.js)
- Any query that does NOT use parameterized bindings / prepared statements
- Missing `?` placeholders or `$1` parameters

**Pass criteria:** 100% of database queries use parameterized bindings.

#### Finding 2: Path Traversal

Search for:
- File download endpoints that accept user-controlled paths
- `fs.readFile`, `readFileSync`, `createReadStream` with user input
- Missing `path.basename()` or path sanitization
- Direct concatenation of user input with file system paths

**Pass criteria:** No user-controlled file paths in server-side code, or all paths sanitized with basename/allowlist.

#### Finding 3: Race Conditions

Search for:
- Check-then-act patterns (SELECT then UPDATE/INSERT)
- Payment/webhook handlers without idempotency guards
- Credit/balance operations without atomic conditional updates
- Missing database transactions or batch operations

**Pass criteria:** All financial operations use atomic patterns (conditional UPDATE, INSERT OR IGNORE + UNIQUE, or database transactions).

#### Finding 4: Hardcoded Secrets

Search for:
- API keys, passwords, tokens written directly in source code
- `.env` files committed to git
- Secrets in git history
- Check `.gitignore` covers `.env*` files

**Pass criteria:** Zero secrets in source code. All secrets via environment variables or secret managers.

#### Finding 5: Audit Logging

Check for:
- Structured logging on all sensitive operations
- Timestamps on all transaction records
- Security event logging (failed auth, rejected requests, rate limits)
- Log persistence (not just ephemeral console.log)

**Pass criteria:** All sensitive operations produce traceable audit entries with timestamps.

#### Finding 6: Testing Before Go-Live

Check for:
- Test suite exists and passes
- CI/CD pipeline gates deployment on test passage
- Security-specific tests (auth boundaries, input validation)
- Dependency vulnerability scanning in CI

**Pass criteria:** Tests exist, run in CI, and block deployment on failure.

#### Finding 7: Dependency Safety

Check for:
- Suspicious or typosquatting package names
- Packages without clear provenance
- Missing lockfile
- Malicious postinstall scripts
- Unpinned versions without lockfile

**Pass criteria:** All dependencies are well-known, lockfile exists, no suspicious packages.

#### Finding 8: License Compliance

Check for:
- GPL-licensed code copied into source without compliance
- Removed copyright notices
- License file exists for the project
- node_modules not committed

**Pass criteria:** Project has a license, no GPL code copied without compliance, copyright notices preserved.

### Phase 2: OWASP Top 10 Assessment

For each category, check controls and identify gaps:

| Category | Key Checks |
|----------|-----------|
| A01 Broken Access Control | Auth on all protected routes, RBAC/ABAC, tenant-scoped data, no IDOR |
| A02 Cryptographic Failures | HTTPS enforced, secrets server-side only, PII encrypted at rest, timing-safe comparisons |
| A03 Injection | Parameterized queries, schema validation, no command execution, CSP configured |
| A04 Insecure Design | Rate limiting, idempotency, atomic transactions, input validation |
| A05 Security Misconfiguration | Security headers, no debug modes, sanitized errors, no CORS wildcard |
| A06 Vulnerable Components | Dependency audit passes, lockfile exists, typosquatting checked, known CVEs tracked |
| A07 Auth Failures | Password hashing, session management, brute-force protection, rate limiting |
| A08 Data Integrity | CI gates (SAST, secret scanning), webhook signature verification |
| A09 Logging & Monitoring | Structured audit logs, tamper-resistant storage, alerting |
| A10 SSRF | No raw user-controlled server-side fetch, private IP/metadata blocking, redirect validation |

### Phase 2.5: Enterprise Remediation Routing

When producing the fix list, map each finding to the dedicated remediation skill:

| Finding | Route to |
|---------|----------|
| Broken access control, IDOR, tenant leaks, admin exposure | `$access-control-patterns` |
| SQL injection, raw queries, unsafe DB calls | `$sql-injection-prevention` |
| XSS, path traversal, command injection, unsafe file uploads | `$input-validation` |
| Hardcoded secrets, committed `.env`, secret rotation | `$secrets-management` |
| Missing CSP/HSTS/clickjacking headers | `$security-headers` |
| Weak passwords, JWT misuse, session fixation, brute-force | `$auth-hardening` |
| CORS wildcard, credential leakage, unsafe preflight | `$cors-configuration` |
| Plaintext PII, missing encryption, weak key management | `$data-encryption` |
| Stack trace leaks, verbose production errors | `$error-handling-security` |
| Missing audit trail or non-persistent logs | `$audit-logging-workers` |
| Webhook replay, race conditions, missing idempotency | `$webhook-security` |
| Missing SAST, secret scanning, dependency gates | `$ci-security-gates` |
| Missing API abuse limits or brute-force limits | `$rate-limiting` |
| SSRF, user-controlled URLs, metadata endpoint exposure | `$ssrf-prevention` |
| Typosquatting, malicious packages, missing lockfile | `$dependency-lockfile-audit` |

### Phase 3: Output

Produce a scorecard:

```markdown
## PDPA Forensic Findings

| # | Finding | Status | Evidence |
|---|---------|--------|----------|
| 1 | SQL Injection | PASS/FAIL | [file:line] |
| 2 | Path Traversal | PASS/FAIL | [file:line] |
| 3 | Race Conditions | PASS/FAIL | [file:line] |
| 4 | Hardcoded Secrets | PASS/FAIL | [file:line] |
| 5 | Audit Logging | PASS/FAIL | [file:line] |
| 6 | Testing | PASS/FAIL | [file:line] |
| 7 | Dependencies | PASS/FAIL | [file:line] |
| 8 | License | PASS/FAIL | [file:line] |

## OWASP Top 10

| Category | Rating | Key Finding |
|----------|--------|-------------|
| A01-A10 | Strong/Good/Weak | ... |

## Priority Fixes

1. [Highest risk item first]
2. ...
```

## Legal Reference

- **PDPA 2010 (Pindaan 2024) Seksyen 5(1)(2):** Denda sehingga RM1,000,000 atau penjara sehingga 3 tahun
- **Akta Hak Cipta 1987 Seksyen 41(1)(i):** Denda RM2,000-RM20,000 per fail atau penjara 5 tahun
- **Akta Kontrak 1950 Seksyen 74:** Ganti rugi am dan khas
- **Kanun Keseksaan Seksyen 417/419:** Penjara 5-7 tahun (penipuan)

## Important Notes

- This is an automated internal assessment, not a substitute for external penetration testing
- For enterprise/government contracts, commission a CREST-certified external pentest
- Security is maintenance, not a one-time task -- re-run this audit after major changes
