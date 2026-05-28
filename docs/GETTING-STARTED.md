# Getting Started for Vibe Coders

This guide is for absolute beginners.

Prefer Bahasa Melayu daily tone? Read: **[Panduan Mula Untuk Vibe Coder](./PANDUAN-BM.md)**

If you are building an app with an AI coding agent and you are not a security expert, start here.

## What This Is

`security-for-vibecoders` is a security skill pack for AI coding agents.

That means you install it once, then your AI agent can use it to check your code for common security problems.

Examples of agents:

- Claude Code
- OpenCode
- Cursor
- Windsurf
- Codex

Think of this like giving your AI agent a security checklist and a set of repair instructions.

Instead of asking:

```text
Is my app secure?
```

You can ask:

```text
Run a PDPA and OWASP security audit on this codebase using secure-ship. Explain the findings in beginner language, then fix the high-risk issues first.
```

## Why You Need This

When you vibe code, the app may work, but it may still be unsafe.

Common beginner security mistakes include:

- Putting API keys directly inside code
- Letting users access other users' data
- Building login without rate limiting
- Using raw SQL that can be hacked
- Showing full error messages to users
- Accepting file uploads without checking the file
- Forgetting audit logs
- Deploying without security checks

This skill pack helps your agent find and fix those problems before you ship.

## What It Checks

The full audit checks for:

- SQL injection
- Broken access control
- IDOR bugs, where one user can access another user's record
- Missing tenant isolation for SaaS apps
- Hardcoded secrets and leaked API keys
- Weak authentication
- Unsafe CORS settings
- Missing security headers
- Unsafe file uploads
- Unsafe error messages
- Missing audit logs
- Webhook race conditions
- Missing rate limits
- SSRF bugs from fetching user-provided URLs
- Suspicious or vulnerable dependencies
- Missing CI security gates
- PDPA 2024 security expectations for Malaysian apps
- OWASP Top 10 application security risks

## Before You Install

You need these basics first:

- A code project on your computer
- Node.js installed
- An AI coding agent installed, such as Claude Code, OpenCode, Cursor, Windsurf, or Codex
- Terminal access

If you do not know whether Node.js is installed, open your terminal and run:

```bash
node -v
```

If you see a version like `v20.11.0`, Node.js is installed.

If the command fails, install Node.js from:

```text
https://nodejs.org
```

## Installation

Open your terminal inside your app project.

For example, if your app is in a folder called `my-saas-app`, open that folder in your terminal first.

Then run:

```bash
npx skills add afu-it/secure-ship
```

This installs the security skills into the current project.

If you want the skill available in all projects, run:

```bash
npx skills add afu-it/secure-ship -g
```

If you only want to preview what will be installed, run:

```bash
npx skills add afu-it/secure-ship --list
```

## Which Install Command Should I Use?

Use this if you only want security skills in one project:

```bash
npx skills add afu-it/secure-ship
```

Use this if you want security skills available everywhere:

```bash
npx skills add afu-it/secure-ship -g
```

For most beginners, project install is safer:

```bash
npx skills add afu-it/secure-ship
```

## How To Use It

After installation, open your AI coding agent inside your project.

Then copy and paste one of the prompts below.

## Best First Prompt

Use this when you want a full security scan:

```text
Run a full security audit on this codebase using the secure-ship skill.

Check PDPA 2024 Malaysia compliance and OWASP Top 10 issues.

Explain every finding in beginner-friendly language.

Group findings by risk level: Critical, High, Medium, Low.

Do not fix anything yet. First show me the report and ask me what to fix.
```

This is the safest first prompt because the agent only reports problems. It does not change your code yet.

## Prompt To Scan And Fix

Use this when you want the agent to scan and fix serious issues:

```text
Run a full security audit using secure-ship.

Check PDPA 2024 Malaysia and OWASP Top 10.

Fix only Critical and High risk issues first.

Keep changes small and explain each fix in simple beginner language.

After fixing, run the relevant tests or checks and show me what passed.
```

## Prompt Before Deployment

Use this before you publish your app:

```text
I am about to deploy this app to production.

Use secure-ship to run a pre-deployment security review.

Check for hardcoded secrets, SQL injection, broken access control, unsafe CORS, missing security headers, weak auth, missing rate limits, unsafe errors, missing audit logs, and dependency risks.

Give me a go-live checklist with pass/fail status.
```

## Prompt For SaaS Apps

Use this if your app has teams, tenants, organizations, companies, or workspaces:

```text
Use secure-ship to check access control and tenant isolation.

Make sure one user, team, organization, or tenant cannot access another tenant's data.

Look for IDOR bugs, missing userId filters, missing tenantId filters, and weak permission checks.

Explain findings in simple language and suggest safe fixes.
```

## Prompt For SQL Injection

Use this if your app uses a database:

```text
Use secure-ship to check for SQL injection.

Find raw SQL, string concatenation in queries, unsafe filters, unsafe search, and unsafe order-by logic.

Replace unsafe queries with parameterized queries or safe ORM patterns.
```

## Prompt For Hardcoded Secrets

Use this if you pasted API keys into your code before:

```text
Use secure-ship to check for hardcoded secrets.

Look for API keys, database passwords, JWT secrets, webhook secrets, private keys, tokens, and credentials in source code.

Move secrets to environment variables.

Create or update .env.example without real secret values.

Make sure .env is ignored by git.
```

## Prompt For Authentication

Use this if your app has login or signup:

```text
Use secure-ship to harden authentication.

Check password hashing, session security, JWT expiry, refresh tokens, login rate limiting, brute-force protection, password reset flow, and secure cookies.

Explain weak points in beginner language before fixing.
```

## Prompt For API Routes

Use this if your app has backend APIs:

```text
Use secure-ship to audit all API routes.

Check that protected routes require authentication.

Check that users can only access their own data.

Check input validation, rate limiting, CORS, error handling, audit logs, and security headers.

Give me a route-by-route report.
```

## Prompt For File Uploads

Use this if users can upload files:

```text
Use secure-ship to audit file upload security.

Check file type validation, file size limits, path traversal, unsafe filenames, malware risk, public file access, and storage permissions.

Do not trust file extensions only. Check magic bytes where possible.
```

## Prompt For Webhooks

Use this if you use Stripe, Billplz, ToyyibPay, Midtrans, PayPal, or any payment webhook:

```text
Use secure-ship to audit webhook security.

Check signature verification, timing-safe comparison, idempotency, race conditions, duplicate payment handling, replay attacks, and audit logs.

Fix high-risk webhook issues first.
```

## Prompt For Dependencies

Use this if you installed many npm packages:

```text
Use secure-ship to audit dependencies.

Check for vulnerable packages, suspicious packages, typosquatting, lockfile tampering, abandoned packages, and unsafe install scripts.

Tell me which packages are risky and what to replace or update.
```

## Prompt For CI Security Checks

Use this if you use GitHub:

```text
Use secure-ship to add security checks to CI.

Add dependency audit, secret scanning, and static analysis where appropriate.

Prefer GitHub Actions.

Keep the workflow simple enough for a beginner to understand.
```

## How To Talk To The Agent

Good prompts are specific.

Bad prompt:

```text
Make secure.
```

Better prompt:

```text
Use secure-ship to audit my API routes for broken access control, SQL injection, input validation, CORS, and unsafe error messages. Show findings first. Do not fix until I approve.
```

Best prompt:

```text
Use secure-ship to audit this Next.js SaaS app before production. It has users, teams, Stripe payments, file uploads, and admin routes. Check PDPA 2024 Malaysia and OWASP Top 10. Produce a beginner-friendly report with exact files, risk level, why it matters, and recommended fix. Do not modify code yet.
```

## What The Report Means

A good security report should include:

- File path
- Line number if possible
- Risk level
- What is wrong
- Why it is dangerous
- How to fix it
- Whether the agent can fix it safely

Example:

```text
High Risk: Broken access control

File: app/api/invoices/[id]/route.ts

Problem: The route checks that the user is logged in, but it does not check whether the invoice belongs to that user.

Why this matters: A user could change the invoice ID in the URL and read another user's invoice.

Fix: Add a userId or tenantId filter to the database query.
```

## Risk Levels Explained

Critical means someone may be able to steal data, bypass login, access another user's account, leak secrets, or take over the app.

High means the bug is dangerous and should be fixed before production.

Medium means the bug may become dangerous depending on how the app is used.

Low means it is still worth fixing, but usually not an emergency.

## Safe Beginner Workflow

Use this workflow if you are new:

1. Ask the agent to scan only.
2. Read the report.
3. Ask the agent to fix Critical issues first.
4. Run tests.
5. Ask the agent to fix High issues.
6. Run tests again.
7. Ask the agent to explain what changed.
8. Commit your code.
9. Deploy only after the important issues are fixed.

Copy this prompt:

```text
Use secure-ship in beginner-safe mode.

Step 1: Scan only. Do not edit files.

Step 2: Explain findings simply.

Step 3: Ask me before fixing.

When I approve fixes, fix only one risk category at a time and run tests after each batch.
```

## What Not To Do

Do not paste real secrets into the AI chat.

Do not ask the agent to ignore security warnings just to deploy faster.

Do not fix everything in one giant change if you are a beginner.

Do not deploy if Critical or High findings are still open.

Do not assume the app is safe just because it works in the browser.

## Beginner Security Checklist

Before production, ask your agent to confirm:

- No hardcoded API keys
- `.env` is not committed
- `.env.example` exists with fake values
- Login routes have rate limiting
- Passwords are hashed securely
- Protected routes require authentication
- Users cannot access other users' data
- Admin routes require admin permission
- Database queries are parameterized
- User input is validated
- File uploads are checked safely
- CORS does not allow every website with credentials
- Security headers are enabled
- Error messages do not leak stack traces
- Webhooks verify signatures
- Payment webhooks are idempotent
- Audit logs exist for sensitive actions
- Dependencies are checked
- Security checks run in CI

## Common Problems

### `npx skills` command not found

Make sure Node.js is installed:

```bash
node -v
```

Then try again:

```bash
npx skills add afu-it/secure-ship
```

### The agent does not use the skill

Be explicit in your prompt:

```text
Use the secure-ship skill to run this security audit.
```

### The agent wants to edit too many files

Tell it to slow down:

```text
Do not fix everything at once. Fix only Critical issues first. Keep changes small. Ask before continuing.
```

### The report is too technical

Ask for beginner language:

```text
Explain this like I am a beginner. Tell me what can go wrong, how serious it is, and what the fix does.
```

### I am scared to break my app

Use scan-only mode:

```text
Run the security scan only. Do not edit files. Show me the report first.
```

## Copy-Paste Starter Pack

If you only use one prompt, use this:

```text
Use secure-ship to run a full beginner-friendly security audit on this codebase.

Check PDPA 2024 Malaysia and OWASP Top 10.

Look for SQL injection, broken access control, IDOR, tenant isolation bugs, hardcoded secrets, weak auth, unsafe CORS, missing security headers, unsafe file uploads, unsafe error messages, missing audit logs, webhook bugs, missing rate limits, SSRF, and dependency risks.

Do not edit files yet.

Create a report with:
- risk level
- file path
- problem
- why it matters
- simple fix recommendation
- whether you can fix it safely

After the report, ask me which issues to fix first.
```

## After The Scan

If the agent finds issues, respond with:

```text
Fix the Critical issues first.

Keep changes minimal.

Explain each file you change.

Run tests or checks after fixing.
```

Then after Critical issues are fixed:

```text
Now fix the High risk issues.

Keep changes minimal.

Run tests after fixing.
```

## Final Go-Live Prompt

Use this right before deployment:

```text
Use secure-ship to create a final go-live security checklist.

Mark each item as Pass, Fail, or Not Applicable.

If anything is Critical or High risk, tell me not to deploy yet.

Explain the remaining risks in beginner language.
```

## Simple Mental Model

Your app is not secure because it looks finished.

Your app is safer when:

- Users can only access their own data
- Secrets are not inside code
- Inputs are checked before use
- Database queries cannot be injected
- Login cannot be brute-forced
- Errors do not reveal internals
- Important actions are logged
- Security checks run before deployment

This skill pack helps your AI agent check those things for you.
