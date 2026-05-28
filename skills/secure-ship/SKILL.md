---
name: secure-ship
description: "Security for vibe coders. Full PDPA 2024 + OWASP Top 10 compliance for any codebase. Covers enterprise-grade access control, SSRF prevention, SQL injection, input validation, secrets management, auth hardening, security headers, CORS, data encryption, error handling, audit logging, webhook security, CI/CD gates, rate limiting, and dependency auditing. Based on real Malaysian court cases where developers were fined RM1,000,000. Use when asked to secure a project, check PDPA compliance, run OWASP audit, add security to CI, or before go-live."
version: "2.1"
---

# secure-ship

Security skill pack for developers shipping production code. Based on real forensic findings from Malaysian court cases under PDPA 2024.

## When to Activate

- User asks to "secure my project" or "check security"
- User mentions PDPA, OWASP, or security audit
- User is about to deploy to production
- User asks about SQL injection, webhooks, rate limiting, or audit logs
- User says "is my code safe?" or "can I get sued?"

## Sub-Skills

Route to the appropriate sub-skill based on the request:

| Request | Route to |
|---------|----------|
| "Run a security audit" / "Check PDPA compliance" / "OWASP assessment" | `$pdpa-security-audit` |
| "Access control" / "IDOR" / "tenant isolation" / "RBAC" / "permissions" | `$access-control-patterns` |
| "Check for SQL injection" / "Fix my queries" / "parameterized" | `$sql-injection-prevention` |
| "Validate input" / "XSS" / "path traversal" / "sanitize" / "file upload" | `$input-validation` |
| "Fix secrets" / "hardcoded API key" / ".env setup" / "secret rotation" | `$secrets-management` |
| "Security headers" / "CSP" / "HSTS" / "clickjacking" / "helmet" | `$security-headers` |
| "Fix auth" / "password hashing" / "JWT" / "session" / "brute force" | `$auth-hardening` |
| "CORS" / "cross-origin" / "preflight" / "Access-Control" | `$cors-configuration` |
| "Encrypt data" / "encrypt PII" / "AES" / "field encryption" / "PDPA encryption" | `$data-encryption` |
| "Error leaking" / "stack trace" / "verbose errors" / "hide errors" | `$error-handling-security` |
| "Add audit logging" / "I need logs" / "PDPA audit trail" | `$audit-logging-workers` |
| "Fix webhook" / "race condition" / "double credit" / "idempotency" | `$webhook-security` |
| "Add security to CI" / "SAST" / "secret scanning" / "CodeQL" | `$ci-security-gates` |
| "Add rate limiting" / "prevent abuse" / "too many requests" | `$rate-limiting` |
| "SSRF" / "fetch user URL" / "URL preview" / "metadata endpoint" / "egress" | `$ssrf-prevention` |
| "Check dependencies" / "typosquatting" / "malicious package" / "lockfile" | `$dependency-lockfile-audit` |
| General "secure my project" | Run `$pdpa-security-audit` first, then fix findings using other sub-skills |

## Quick Audit Checklist

When a user asks "is my code secure?", run through this checklist:

```
[ ] All DB queries use parameterized bindings (no string concatenation)
[ ] All protected routes enforce authentication and authorization
[ ] All tenant-owned queries are scoped by tenantId/userId
[ ] IDOR tests cover cross-tenant access attempts
[ ] All user input validated with schema (Zod/Joi/Pydantic)
[ ] No hardcoded secrets in source code
[ ] .env files are gitignored, .env.example committed
[ ] Passwords hashed with Argon2id or bcrypt (cost 12+)
[ ] Auth enforced on all protected routes
[ ] Login rate-limited (brute-force protection)
[ ] JWTs expire in 15 minutes, refresh tokens in httpOnly cookies
[ ] CORS whitelist specific origins (no wildcard with credentials)
[ ] PII fields encrypted at rest (AES-256-GCM)
[ ] Security headers configured (CSP, HSTS, X-Frame-Options)
[ ] Error responses don't leak stack traces or SQL queries
[ ] Payment webhooks have idempotency guards
[ ] Webhook tokens use timing-safe comparison
[ ] Structured audit logs on sensitive operations
[ ] Tests exist and run in CI before deploy
[ ] npm audit / dependency scanning in CI
[ ] SAST (CodeQL/Semgrep) in CI
[ ] Secret scanning (Gitleaks) in CI
[ ] Lockfile committed and verified with npm ci
[ ] No typosquatted or suspicious dependencies
[ ] Rate limiting on authenticated endpoints
[ ] File uploads validate magic bytes, not just extension
[ ] No user input passed to shell commands or eval()
[ ] No raw server-side fetch of user-provided URLs
[ ] SSRF controls block private, loopback, link-local, and metadata IPs
[ ] License file exists, no GPL violations
```

If any item fails, route to the appropriate sub-skill to fix it.

## Legal Context

This skill pack exists because of PDPA 2010 (Pindaan 2024) which criminalizes negligent Data Processors:

- **Seksyen 5(1)(2):** RM1,000,000 fine or 3 years prison for security failures
- **Akta Hak Cipta 1987:** RM2,000-RM20,000 per file for copyright violations
- **Kanun Keseksaan 417/419:** 5-7 years for fraud/misrepresentation

The standard of care is: "Did you implement security measures proportionate to the data you handle?" These skills help you demonstrate that standard was met.
