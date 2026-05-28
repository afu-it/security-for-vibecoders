---
name: ci-security-gates
description: "Add CodeQL SAST, Gitleaks secret scanning, and dependency audit to any CI/CD pipeline. Blocks deployment if vulnerabilities or secrets are detected. Covers GitHub Actions, GitLab CI, and generic pipelines. Use when setting up CI security, adding SAST, preventing secret leaks, or hardening deployment pipelines."
version: "1.0"
---

# CI/CD Security Gates

Add automated security scanning to your CI/CD pipeline. Blocks deployment if code vulnerabilities, leaked secrets, or dangerous dependencies are detected.

## Why This Matters

Finding #6 from Malaysian court cases: "No evidence the system went through testing or vulnerability scanning before go-live."

Under PDPA 2024, shipping untested code with known vulnerabilities is criminal negligence.

## The Three Gates

| Gate | What it catches | Tool |
|------|----------------|------|
| **SAST** | SQL injection, XSS, insecure patterns in YOUR code | CodeQL / Semgrep |
| **Secret Scanning** | API keys, passwords, tokens accidentally committed | Gitleaks |
| **Dependency Audit** | Known CVEs in third-party packages | npm audit / pip audit |

## GitHub Actions Implementation

### Full Security Pipeline

```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  security:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    steps:
      - name: Checkout
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript-typescript
          queries: security-and-quality

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: /language:javascript-typescript

      - name: Run Gitleaks secret scanning
        uses: gitleaks/gitleaks-action@44c470fbc1e0e0de7e9ddc8d76cd120e42694ebb # v2.3.8
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  deploy:
    needs: security  # BLOCKS deploy if security fails
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Setup Node
        uses: actions/setup-node@49933ea5288caeca8642195f572a2b2b8a0a4b0e # v4.4.0
        with:
          node-version: 22
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm run test

      - name: Dependency security audit
        run: npm audit --audit-level=high

      - name: Build
        run: npm run build

      - name: Deploy
        run: npm run deploy
```

### Key Principles

1. **Pin actions to commit SHAs** -- prevents supply-chain attacks via tag mutation
2. **Security job runs BEFORE deploy** -- `needs: security` blocks deployment
3. **Use `npm ci`** -- installs from lockfile deterministically
4. **Audit at `high` level** -- blocks on high/critical CVEs, reports moderate

## SHA Pinning

Never use mutable tags like `@v4`. Always pin to commit SHA:

```yaml
# BAD -- tag can be moved to malicious commit
uses: actions/checkout@v4

# GOOD -- immutable reference
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

How to find the SHA:
1. Go to the action's GitHub releases page
2. Find the version tag (e.g., v4.2.2)
3. Click the commit hash
4. Copy the full 40-character SHA

## CodeQL Configuration

CodeQL catches:
- SQL injection patterns
- Cross-site scripting (XSS)
- Path traversal
- Insecure randomness
- Hard-coded credentials
- Prototype pollution
- Command injection

Supported languages: `javascript-typescript`, `python`, `java`, `go`, `ruby`, `csharp`, `cpp`

## Gitleaks Configuration

Gitleaks catches:
- AWS access keys
- Google Cloud keys
- Stripe/Xendit/payment API keys
- Database connection strings
- Private keys (RSA, EC)
- JWT secrets
- Generic high-entropy strings

### Custom Rules (`.gitleaks.toml`)

```toml
[extend]
useDefault = true

[[rules]]
id = "clerk-secret-key"
description = "Clerk Secret Key"
regex = '''sk_(test|live)_[A-Za-z0-9]+'''
tags = ["key", "clerk"]

[[rules]]
id = "xendit-secret-key"
description = "Xendit Secret Key"
regex = '''xnd_(development|production)_[A-Za-z0-9]+'''
tags = ["key", "xendit"]

[allowlist]
paths = [
  '''.env.example''',
  '''.*test.*''',
]
```

## Dependency Audit

### Node.js
```bash
# In CI -- fail on high/critical
npm audit --audit-level=high

# Local -- see everything
npm audit

# Fix what's safe to fix
npm audit fix
```

### Python
```bash
pip audit --fix --dry-run
pip audit --desc
```

### Package.json Scripts

```json
{
  "scripts": {
    "security:audit": "npm audit --audit-level=high",
    "security:test": "npm run test && npm run security:audit",
    "verify": "npm run test && npm run security:audit && npm run build"
  }
}
```

## GitLab CI Equivalent

```yaml
stages:
  - security
  - test
  - deploy

sast:
  stage: security
  image: returntocorp/semgrep
  script:
    - semgrep --config=auto --error .
  allow_failure: false

secret-detection:
  stage: security
  image: zricethezav/gitleaks
  script:
    - gitleaks detect --source . --verbose
  allow_failure: false

test:
  stage: test
  needs: [sast, secret-detection]
  script:
    - npm ci
    - npm run test
    - npm audit --audit-level=high

deploy:
  stage: deploy
  needs: [test]
  script:
    - npm run deploy
  only:
    - main
```

## What Happens When Gates Fail

| Gate | Failure | Action |
|------|---------|--------|
| CodeQL | SQL injection pattern found | Fix the query, use parameterized binding |
| Gitleaks | Secret detected in commit | Remove secret, rotate it, add to .gitignore |
| npm audit | High/critical CVE in dependency | Update the package or find alternative |
| Tests | Test failure | Fix the code before deploying |

## PDPA 2024 Compliance

This satisfies:
- **Evidence of due diligence** -- automated scanning proves you checked before shipping
- **Continuous monitoring** -- runs on every push, not just once
- **Proportionate security measures** -- SAST + secret scanning + dependency audit covers the major vectors
- **Audit trail** -- GitHub Actions logs provide timestamped evidence of every security check

If JPDP asks "did you test for vulnerabilities before go-live?", you can point to the CI pipeline history showing every commit was scanned.
