# Panduan Mula Untuk Vibe Coder

Panduan ni untuk beginner yang baru belajar bina app dengan AI coding agent.

Kalau app korang dah boleh jalan, belum tentu app tu selamat. Guide ni ajar cara guna `security-for-vibecoders` supaya AI agent boleh tolong buat `security audit`, cari masalah, dan fix benda penting sebelum deploy.

Technical terms macam `security audit`, `SQL injection`, `hardcoded secrets`, `rate limiting`, `OWASP`, dan `PDPA` dikekalkan dalam English supaya senang cari balik dalam code, docs, dan error message.

## Apa Benda Ni?

`security-for-vibecoders` ialah security skill pack untuk AI coding agent.

Maksudnya, korang install skill ni, lepas tu AI agent korang boleh guna checklist dan pattern security untuk scan code korang.

Contoh AI coding agent:

- Claude Code
- OpenCode
- Cursor
- Windsurf
- Codex

Senang cerita, skill ni macam bagi AI agent korang otak security tambahan.

Daripada tanya benda umum macam:

```text
App aku secure tak?
```

Tanya macam ni:

```text
Run a PDPA and OWASP security audit on this codebase using secure-ship. Explain the findings in beginner language, then fix the high-risk issues first.
```

## Kenapa Korang Perlu Guna Ni?

Bila vibe code, AI boleh buat app laju. Tapi AI juga boleh buat security mistake tanpa korang perasan.

Masalah biasa beginner:

- Letak API key terus dalam code
- User boleh access data user lain
- Login tak ada `rate limiting`
- Query database guna raw SQL yang boleh kena `SQL injection`
- Error message tunjuk stack trace atau database detail
- File upload tak check file betul-betul
- Tak ada `audit logs`
- Deploy tanpa `security checks`

Skill ni bantu AI agent korang cari dan fix masalah macam ni sebelum production.

## Apa Yang Dia Check?

Full audit akan check benda ni:

- `SQL injection`
- Broken access control
- `IDOR` bugs, iaitu user boleh access record user lain
- Missing tenant isolation untuk SaaS app
- `Hardcoded secrets` dan leaked API keys
- Weak authentication
- Unsafe CORS settings
- Missing security headers
- Unsafe file uploads
- Unsafe error messages
- Missing audit logs
- Webhook race conditions
- Missing rate limits
- `SSRF` bugs bila app fetch URL daripada user
- Suspicious atau vulnerable dependencies
- Missing CI security gates
- `PDPA 2024` security expectations untuk app Malaysia
- `OWASP Top 10` application security risks

## Sebelum Install

Pastikan korang ada benda ni dulu:

- Project code dalam komputer
- Node.js installed
- AI coding agent macam Claude Code, OpenCode, Cursor, Windsurf, atau Codex
- Terminal access

Kalau tak tahu Node.js dah install ke belum, buka terminal dan run:

```bash
node -v
```

Kalau keluar version macam `v20.11.0`, maksudnya Node.js dah ada.

Kalau command tu error, install Node.js dari:

```text
https://nodejs.org
```

## Cara Install

Buka terminal dalam folder app korang.

Contoh, kalau app korang dalam folder `my-saas-app`, buka terminal dekat folder tu.

Lepas tu run:

```bash
npx skills add afu-it/secure-ship
```

Command ni install security skills dalam current project.

Kalau nak skill ni available untuk semua project, run:

```bash
npx skills add afu-it/secure-ship -g
```

Kalau nak tengok dulu apa yang akan install:

```bash
npx skills add afu-it/secure-ship --list
```

## Command Mana Patut Guna?

Kalau beginner, guna yang ni dulu:

```bash
npx skills add afu-it/secure-ship
```

Sebab dia cuma install dekat project sekarang. Lebih senang control.

Kalau korang memang nak guna untuk semua project:

```bash
npx skills add afu-it/secure-ship -g
```

## Cara Guna Lepas Install

Lepas install, buka AI coding agent dalam project korang.

Lepas tu copy paste prompt yang sesuai bawah ni.

## Prompt Pertama Yang Paling Safe

Guna prompt ni kalau korang nak scan dulu tanpa ubah code:

```text
Run a full security audit on this codebase using the secure-ship skill.

Check PDPA 2024 Malaysia compliance and OWASP Top 10 issues.

Explain every finding in beginner-friendly language.

Group findings by risk level: Critical, High, Medium, Low.

Do not fix anything yet. First show me the report and ask me what to fix.
```

Ini cara paling safe untuk mula sebab agent hanya buat report. Dia tak edit file lagi.

## Prompt Untuk Scan Dan Fix

Guna ni bila korang dah ready agent fix issue serius:

```text
Run a full security audit using secure-ship.

Check PDPA 2024 Malaysia and OWASP Top 10.

Fix only Critical and High risk issues first.

Keep changes small and explain each fix in simple beginner language.

After fixing, run the relevant tests or checks and show me what passed.
```

## Prompt Sebelum Deploy

Guna ni sebelum publish app:

```text
I am about to deploy this app to production.

Use secure-ship to run a pre-deployment security review.

Check for hardcoded secrets, SQL injection, broken access control, unsafe CORS, missing security headers, weak auth, missing rate limits, unsafe errors, missing audit logs, and dependency risks.

Give me a go-live checklist with pass/fail status.
```

## Prompt Untuk SaaS App

Guna ni kalau app korang ada team, tenant, organization, company, workspace, atau role user:

```text
Use secure-ship to check access control and tenant isolation.

Make sure one user, team, organization, or tenant cannot access another tenant's data.

Look for IDOR bugs, missing userId filters, missing tenantId filters, and weak permission checks.

Explain findings in simple language and suggest safe fixes.
```

## Prompt Untuk SQL Injection

Guna ni kalau app korang guna database:

```text
Use secure-ship to check for SQL injection.

Find raw SQL, string concatenation in queries, unsafe filters, unsafe search, and unsafe order-by logic.

Replace unsafe queries with parameterized queries or safe ORM patterns.
```

## Prompt Untuk Hardcoded Secrets

Guna ni kalau korang pernah paste API key, token, atau password dalam code:

```text
Use secure-ship to check for hardcoded secrets.

Look for API keys, database passwords, JWT secrets, webhook secrets, private keys, tokens, and credentials in source code.

Move secrets to environment variables.

Create or update .env.example without real secret values.

Make sure .env is ignored by git.
```

## Prompt Untuk Authentication

Guna ni kalau app ada login atau signup:

```text
Use secure-ship to harden authentication.

Check password hashing, session security, JWT expiry, refresh tokens, login rate limiting, brute-force protection, password reset flow, and secure cookies.

Explain weak points in beginner language before fixing.
```

## Prompt Untuk API Routes

Guna ni kalau app ada backend API:

```text
Use secure-ship to audit all API routes.

Check that protected routes require authentication.

Check that users can only access their own data.

Check input validation, rate limiting, CORS, error handling, audit logs, and security headers.

Give me a route-by-route report.
```

## Prompt Untuk File Upload

Guna ni kalau user boleh upload file:

```text
Use secure-ship to audit file upload security.

Check file type validation, file size limits, path traversal, unsafe filenames, malware risk, public file access, and storage permissions.

Do not trust file extensions only. Check magic bytes where possible.
```

## Prompt Untuk Webhook

Guna ni kalau korang pakai Stripe, Billplz, ToyyibPay, Midtrans, PayPal, atau payment webhook lain:

```text
Use secure-ship to audit webhook security.

Check signature verification, timing-safe comparison, idempotency, race conditions, duplicate payment handling, replay attacks, and audit logs.

Fix high-risk webhook issues first.
```

## Prompt Untuk Dependencies

Guna ni kalau korang install banyak npm package:

```text
Use secure-ship to audit dependencies.

Check for vulnerable packages, suspicious packages, typosquatting, lockfile tampering, abandoned packages, and unsafe install scripts.

Tell me which packages are risky and what to replace or update.
```

## Prompt Untuk CI Security Checks

Guna ni kalau project korang dekat GitHub:

```text
Use secure-ship to add security checks to CI.

Add dependency audit, secret scanning, and static analysis where appropriate.

Prefer GitHub Actions.

Keep the workflow simple enough for a beginner to understand.
```

## Cara Tanya Agent Dengan Betul

Prompt yang baik kena specific.

Prompt kurang bagus:

```text
Make secure.
```

Prompt lebih bagus:

```text
Use secure-ship to audit my API routes for broken access control, SQL injection, input validation, CORS, and unsafe error messages. Show findings first. Do not fix until I approve.
```

Prompt paling bagus:

```text
Use secure-ship to audit this Next.js SaaS app before production. It has users, teams, Stripe payments, file uploads, and admin routes. Check PDPA 2024 Malaysia and OWASP Top 10. Produce a beginner-friendly report with exact files, risk level, why it matters, and recommended fix. Do not modify code yet.
```

## Cara Baca Security Report

Report yang bagus biasanya ada:

- File path
- Line number kalau boleh
- Risk level
- Apa masalah dia
- Kenapa bahaya
- Cara fix
- Sama ada agent boleh fix dengan selamat

Contoh:

```text
High Risk: Broken access control

File: app/api/invoices/[id]/route.ts

Problem: The route checks that the user is logged in, but it does not check whether the invoice belongs to that user.

Why this matters: A user could change the invoice ID in the URL and read another user's invoice.

Fix: Add a userId or tenantId filter to the database query.
```

Maksud simple dia: route tu tahu user dah login, tapi tak check invoice tu milik siapa. Jadi user boleh teka ID invoice orang lain dan tengok data orang.

## Maksud Risk Level

`Critical` maksudnya orang mungkin boleh curi data, bypass login, access akaun orang lain, leak secrets, atau ambil alih app.

`High` maksudnya bug tu bahaya dan patut fix sebelum production.

`Medium` maksudnya bug tu boleh jadi bahaya bergantung pada cara app digunakan.

`Low` maksudnya masih elok fix, tapi biasanya bukan emergency.

## Workflow Safe Untuk Beginner

Kalau korang baru belajar, ikut flow ni:

1. Suruh agent scan sahaja.
2. Baca report.
3. Suruh agent fix `Critical` issues dulu.
4. Run tests.
5. Suruh agent fix `High` issues.
6. Run tests lagi.
7. Suruh agent explain apa yang berubah.
8. Commit code.
9. Deploy hanya bila issue penting dah settle.

Copy prompt ni:

```text
Use secure-ship in beginner-safe mode.

Step 1: Scan only. Do not edit files.

Step 2: Explain findings simply.

Step 3: Ask me before fixing.

When I approve fixes, fix only one risk category at a time and run tests after each batch.
```

## Benda Jangan Buat

Jangan paste real secrets dalam AI chat.

Jangan suruh agent ignore security warning semata-mata nak cepat deploy.

Jangan fix semua benda dalam satu giant change kalau korang beginner.

Jangan deploy kalau masih ada `Critical` atau `High` findings.

Jangan assume app selamat cuma sebab dia nampak okay dalam browser.

## Beginner Security Checklist

Sebelum production, suruh agent confirm:

- Tak ada hardcoded API keys
- `.env` tidak committed
- `.env.example` wujud dengan fake values
- Login routes ada `rate limiting`
- Passwords di-hash dengan secure
- Protected routes require authentication
- Users tak boleh access data user lain
- Admin routes require admin permission
- Database queries guna parameterized queries
- User input divalidate
- File uploads checked dengan selamat
- CORS tak allow semua website dengan credentials
- Security headers enabled
- Error messages tak leak stack traces
- Webhooks verify signatures
- Payment webhooks idempotent
- Audit logs wujud untuk sensitive actions
- Dependencies checked
- Security checks run dalam CI

## Masalah Biasa

### `npx skills` command not found

Check Node.js dulu:

```bash
node -v
```

Lepas tu try lagi:

```bash
npx skills add afu-it/secure-ship
```

### Agent tak guna skill ni

Tulis prompt dengan jelas:

```text
Use the secure-ship skill to run this security audit.
```

### Agent nak edit terlalu banyak file

Suruh dia slow down:

```text
Do not fix everything at once. Fix only Critical issues first. Keep changes small. Ask before continuing.
```

### Report terlalu technical

Minta explain macam beginner:

```text
Explain this like I am a beginner. Tell me what can go wrong, how serious it is, and what the fix does.
```

### Takut app rosak

Guna scan-only mode:

```text
Run the security scan only. Do not edit files. Show me the report first.
```

## Copy-Paste Starter Pack

Kalau nak guna satu prompt sahaja, guna ni:

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

## Lepas Scan, Nak Buat Apa?

Kalau agent jumpa issues, reply macam ni:

```text
Fix the Critical issues first.

Keep changes minimal.

Explain each file you change.

Run tests or checks after fixing.
```

Lepas `Critical` issues settle:

```text
Now fix the High risk issues.

Keep changes minimal.

Run tests after fixing.
```

## Final Go-Live Prompt

Guna ni betul-betul sebelum deploy:

```text
Use secure-ship to create a final go-live security checklist.

Mark each item as Pass, Fail, or Not Applicable.

If anything is Critical or High risk, tell me not to deploy yet.

Explain the remaining risks in beginner language.
```

## Mental Model Simple

App korang bukan secure sebab dia dah siap.

App korang lebih secure bila:

- Users cuma boleh access data sendiri
- Secrets tak ada dalam code
- Inputs diperiksa sebelum digunakan
- Database queries tak boleh kena injection
- Login tak boleh brute-forced dengan mudah
- Errors tak dedah internal details
- Sensitive actions ada logs
- Security checks run sebelum deploy

Skill pack ni bantu AI agent check benda-benda tu untuk korang.
