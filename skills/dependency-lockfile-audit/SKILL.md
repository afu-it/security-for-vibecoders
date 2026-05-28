---
name: dependency-lockfile-audit
description: "Detect typosquatting, malicious packages, and lockfile tampering in npm, pip, and composer dependencies. Covers lockfile integrity verification, known-malicious package detection, and supply chain attack prevention. Use when auditing dependencies, investigating suspicious packages, or hardening the supply chain."
version: "1.0"
---

# Dependency Lockfile Audit

Detect malicious, typosquatted, and vulnerable dependencies before they compromise your application. Supply chain attacks are increasingly common -- a single malicious package can exfiltrate all your environment variables on install.

## The Rule

**ALWAYS commit lockfiles. ALWAYS audit before installing. NEVER ignore security warnings.**

## Attack Vectors

| Attack | How it works | Example |
|--------|-------------|---------|
| Typosquatting | Package name looks like popular one | `expresss` instead of `express` |
| Dependency confusion | Private package name claimed on public registry | `@company/utils` published by attacker on npm |
| Malicious postinstall | Script runs on `npm install` | Steals `.env`, SSH keys, sends to attacker |
| Protestware | Maintainer adds destructive code in update | Wipes files, shows political messages |
| Abandoned takeover | Attacker gains control of unmaintained package | Injects crypto miner |

## Audit Commands

### npm (Node.js)

```bash
# Check for known vulnerabilities
npm audit

# Check for known vulnerabilities (production only)
npm audit --omit=dev

# Fix automatically where possible
npm audit fix

# List all packages with install scripts (potential risk)
npm query ':attr(scripts, [postinstall])' | jq '.[].name'

# Check lockfile integrity
npm ci  # Fails if lockfile doesn't match package.json
```

### pip (Python)

```bash
# Check for known vulnerabilities
pip-audit

# Check with safety
safety check --full-report

# Pin all dependencies
pip freeze > requirements.txt
```

### composer (PHP/Laravel)

```bash
# Check for known vulnerabilities
composer audit

# Verify lockfile integrity
composer validate --strict
```

## Typosquatting Detection

### Common Patterns to Watch For

```
# Character substitution
lodash → 1odash (L vs 1)
express → expresss (double letter)
react → reacct

# Scope confusion
@babel/core → @bable/core
@types/node → @typs/node

# Hyphen/underscore tricks
cross-env → crossenv
node-fetch → nodefetch
```

### Automated Check Script

```bash
#!/bin/bash
# check-suspicious-deps.sh
# Flags packages with very low download counts or recent publish dates

echo "Checking for suspicious packages..."

# Get all dependencies
DEPS=$(jq -r '.dependencies // {} | keys[]' package.json)
DEPS="$DEPS $(jq -r '.devDependencies // {} | keys[]' package.json)"

for pkg in $DEPS; do
  # Get package info from registry
  INFO=$(curl -s "https://registry.npmjs.org/$pkg")
  
  # Check if package exists
  if echo "$INFO" | jq -e '.error' > /dev/null 2>&1; then
    echo "WARNING: Package '$pkg' not found on registry!"
    continue
  fi
  
  # Check weekly downloads (very low = suspicious)
  DOWNLOADS=$(curl -s "https://api.npmjs.org/downloads/point/last-week/$pkg" | jq '.downloads // 0')
  if [ "$DOWNLOADS" -lt 100 ]; then
    echo "WARNING: '$pkg' has only $DOWNLOADS weekly downloads (possible typosquat)"
  fi
done
```

## Lockfile Integrity

### Why Lockfiles Matter

Without a lockfile, `npm install` can resolve to different versions on different machines. An attacker who publishes a malicious patch version will hit anyone without a lockfile.

### Rules

1. **Always commit lockfiles** (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `composer.lock`, `Pipfile.lock`)
2. **Use `npm ci` in CI/CD** (not `npm install`) -- fails if lockfile is out of sync
3. **Review lockfile diffs in PRs** -- unexpected changes may indicate supply chain attack
4. **Pin exact versions** for critical dependencies:
   ```json
   {
     "dependencies": {
       "stripe": "14.5.0",
       "jsonwebtoken": "9.0.2"
     }
   }
   ```

### Detect Lockfile Tampering

```bash
# In CI pipeline -- verify lockfile matches package.json
npm ci --ignore-scripts  # Install without running scripts first

# Then run scripts explicitly after verification
npm rebuild
```

## CI/CD Integration

### GitHub Actions

```yaml
name: Dependency Audit
on:
  pull_request:
    paths:
      - 'package.json'
      - 'package-lock.json'
      - 'requirements.txt'
      - 'Pipfile.lock'

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check lockfile exists
        run: |
          if [ ! -f package-lock.json ]; then
            echo "ERROR: No lockfile found. Run 'npm install' and commit package-lock.json"
            exit 1
          fi

      - name: Install with integrity check
        run: npm ci --ignore-scripts

      - name: Run security audit
        run: npm audit --audit-level=high

      - name: Check for install scripts
        run: |
          SCRIPTS=$(npm query ':attr(scripts, [postinstall])' 2>/dev/null | jq -r '.[].name' 2>/dev/null)
          if [ -n "$SCRIPTS" ]; then
            echo "Packages with postinstall scripts:"
            echo "$SCRIPTS"
            echo "Review these packages for malicious behavior"
          fi
```

## Safe Dependency Practices

1. **Minimize dependencies** -- fewer deps = smaller attack surface
2. **Prefer well-known packages** -- check download counts, maintainer reputation
3. **Lock versions in production** -- use exact versions, not ranges
4. **Review before updating** -- read changelogs, check for ownership changes
5. **Use `--ignore-scripts` for initial install** -- then audit scripts before running
6. **Set up Dependabot/Renovate** -- automated PRs for security updates
7. **Never run `npm install` from untrusted package.json** -- check scripts first

## Red Flags in a Package

- Published in the last 7 days with a name similar to a popular package
- Very few weekly downloads (< 100) for a "utility" package
- `postinstall` script that runs obfuscated code
- Maintainer email is a disposable address
- Repository URL doesn't match package name
- Single version published (no history)
- Minified source code in the package (hiding malicious code)

## PDPA 2024 Consequence

**Finding #7 from forensic analysis:** Suspicious/unverified dependencies.

If a malicious dependency exfiltrates user data from your application, you are liable under Seksyen 5(1) PDPA 2024 for failing to verify your supply chain. The court considers dependency management part of **langkah keselamatan yang munasabah** (reasonable security measures).
