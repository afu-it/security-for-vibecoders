---
name: secrets-management
description: "Prevent hardcoded secrets in source code. Covers .env setup, .gitignore patterns, secret scanning, runtime secret injection, and rotation strategies for Node.js, Python, Laravel, and Cloudflare Workers. Use when setting up environment variables, fixing exposed secrets, or implementing secret rotation."
version: "1.0"
---

# Secrets Management

Prevent hardcoded API keys, passwords, and tokens from leaking into source control. This is Finding #4 from Malaysian PDPA court cases -- secrets in public repos are treated as gross negligence.

## The Rule

**NEVER commit secrets to source code. ALWAYS use environment variables or secret managers.**

## Detection Patterns

### What Counts as a Secret

- API keys (Stripe, OpenAI, AWS, etc.)
- Database connection strings with passwords
- JWT signing keys
- OAuth client secrets
- Webhook signing secrets
- Private keys (RSA, SSH)
- Service account credentials

### Search Patterns for Auditing

```
# Hardcoded strings that look like secrets
(api[_-]?key|secret|password|token|credential|private[_-]?key)\s*[:=]\s*["'][^"']{8,}["']

# AWS keys
AKIA[0-9A-Z]{16}

# Generic high-entropy strings in assignments
(SECRET|KEY|TOKEN|PASSWORD|CREDENTIAL)\s*=\s*["'][A-Za-z0-9+/=]{20,}["']

# Connection strings with embedded passwords
(mongodb|postgres|mysql|redis):\/\/[^:]+:[^@]+@
```

## Correct Setup

### 1. Create `.env` File (NEVER committed)

```bash
# .env (local development only)
DATABASE_URL=postgres://user:pass@localhost:5432/mydb
STRIPE_SECRET_KEY=sk_test_abc123
JWT_SECRET=your-random-64-char-string
OPENAI_API_KEY=sk-proj-abc123
```

### 2. Create `.env.example` (committed, no real values)

```bash
# .env.example (committed to repo as documentation)
DATABASE_URL=postgres://user:password@localhost:5432/dbname
STRIPE_SECRET_KEY=sk_test_xxx
JWT_SECRET=generate-with-openssl-rand-base64-48
OPENAI_API_KEY=sk-proj-xxx
```

### 3. Add to `.gitignore`

```gitignore
# Secrets - NEVER commit these
.env
.env.local
.env.production
.env.*.local
*.pem
*.key
service-account.json
credentials.json
```

## Framework-Specific Patterns

### Node.js / TypeScript

BAD:
```typescript
const stripe = new Stripe('sk_live_abc123xyz');
const jwt = sign(payload, 'my-super-secret-key');
const db = new Pool({ password: 'production-password-123' });
```

GOOD:
```typescript
import { env } from 'process';

const stripe = new Stripe(env.STRIPE_SECRET_KEY!);
const jwt = sign(payload, env.JWT_SECRET!);
const db = new Pool({ connectionString: env.DATABASE_URL });

// Validate at startup
const required = ['STRIPE_SECRET_KEY', 'JWT_SECRET', 'DATABASE_URL'];
for (const key of required) {
  if (!env[key]) throw new Error(`Missing required env var: ${key}`);
}
```

### Cloudflare Workers

BAD:
```typescript
const API_KEY = 'sk-abc123';

export default {
  async fetch(request: Request) {
    return fetch('https://api.openai.com/v1/chat', {
      headers: { Authorization: `Bearer ${API_KEY}` }
    });
  }
};
```

GOOD:
```typescript
// wrangler.toml - declare bindings
// [vars]
// PUBLIC_URL = "https://myapp.com"
//
// Secrets added via: npx wrangler secret put OPENAI_API_KEY

interface Env {
  OPENAI_API_KEY: string;
  PUBLIC_URL: string;
}

export default {
  async fetch(request: Request, env: Env) {
    return fetch('https://api.openai.com/v1/chat', {
      headers: { Authorization: `Bearer ${env.OPENAI_API_KEY}` }
    });
  }
};
```

### Laravel / PHP

BAD:
```php
'stripe' => [
    'secret' => 'sk_live_abc123xyz',
],
```

GOOD:
```php
// config/services.php
'stripe' => [
    'secret' => env('STRIPE_SECRET_KEY'),
],

// Validate in AppServiceProvider::boot()
$required = ['STRIPE_SECRET_KEY', 'DB_PASSWORD', 'APP_KEY'];
foreach ($required as $key) {
    if (empty(env($key))) {
        throw new \RuntimeException("Missing env: {$key}");
    }
}
```

### Python

BAD:
```python
OPENAI_API_KEY = "sk-proj-abc123"
DATABASE_URL = "postgres://admin:password123@prod-db:5432/app"
```

GOOD:
```python
import os
from dotenv import load_dotenv

load_dotenv()

OPENAI_API_KEY = os.environ["OPENAI_API_KEY"]
DATABASE_URL = os.environ["DATABASE_URL"]

# Pydantic settings (recommended)
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    openai_api_key: str
    database_url: str
    jwt_secret: str

    class Config:
        env_file = ".env"

settings = Settings()
```

## Secret Rotation Strategy

### When to Rotate

- Immediately if a secret was ever committed to git (even if removed later)
- Immediately if a team member leaves
- Every 90 days for production secrets (best practice)
- After any security incident

### How to Rotate Without Downtime

```typescript
// Support both old and new key during rotation window
const VALID_API_KEYS = [
  env.API_KEY_CURRENT,
  env.API_KEY_PREVIOUS, // Remove after 24h
].filter(Boolean);

function validateApiKey(key: string): boolean {
  return VALID_API_KEYS.includes(key);
}
```

## Emergency: Secret Was Committed

If a secret was pushed to git, even briefly:

1. **Rotate the secret immediately** -- generate a new one from the provider
2. **Revoke the old secret** -- don't just rotate, explicitly revoke
3. **Check for unauthorized usage** -- review access logs
4. **Clean git history** (optional, secret is already compromised):
   ```bash
   # Only if repo is private and you caught it fast
   git filter-branch --force --index-filter \
     "git rm --cached --ignore-unmatch .env" \
     --prune-empty --tag-name-filter cat -- --all
   ```
5. **Add to .gitignore** to prevent recurrence

## CI/CD Secret Injection

### GitHub Actions

```yaml
# Use repository secrets, never hardcode
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  STRIPE_SECRET_KEY: ${{ secrets.STRIPE_SECRET_KEY }}
```

### Cloudflare Workers (Wrangler)

```bash
# Set secrets via CLI (never in wrangler.toml)
npx wrangler secret put DATABASE_URL
npx wrangler secret put STRIPE_SECRET_KEY
```

## PDPA 2024 Consequence

**Seksyen 5(1) dan 5(2) PDPA 2010 (Pindaan 2024):**
- Denda sehingga RM1,000,000
- Penjara sehingga 3 tahun

Hardcoded secrets in a public repository is considered **kecuaian melampau** (gross negligence). If those secrets provide access to personal data, the developer is criminally liable.
