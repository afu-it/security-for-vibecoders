---
name: auth-hardening
description: "Harden authentication systems against session fixation, brute-force attacks, and weak password storage. Covers bcrypt/argon2 hashing, JWT best practices, session management, and account lockout. Use when implementing login, fixing auth vulnerabilities, or reviewing authentication code."
version: "1.0"
---

# Auth Hardening

Harden authentication to prevent unauthorized access. Weak auth is OWASP A07 and a direct path to PDPA liability -- if attackers access user data through weak login security, the developer is negligent.

## The Rule

**NEVER roll your own crypto. ALWAYS use proven libraries with secure defaults.**

## Password Hashing

### The Only Acceptable Algorithms

| Algorithm | When to use | Config |
|-----------|------------|--------|
| Argon2id | New projects (recommended) | memory: 64MB, iterations: 3, parallelism: 4 |
| bcrypt | Existing projects, wide support | cost factor: 12+ |
| scrypt | Alternative to Argon2 | N=2^17, r=8, p=1 |

**NEVER use:** MD5, SHA-1, SHA-256 (without salt+stretch), plain text

### Node.js

BAD:
```typescript
import crypto from 'crypto';
const hash = crypto.createHash('sha256').update(password).digest('hex');
```

GOOD:
```typescript
import { hash, verify } from '@node-rs/argon2';

// Hash on registration
const passwordHash = await hash(password, {
  memoryCost: 65536,  // 64MB
  timeCost: 3,
  parallelism: 4,
});

// Verify on login
const isValid = await verify(passwordHash, password);
```

Alternative with bcrypt:
```typescript
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12;
const passwordHash = await bcrypt.hash(password, SALT_ROUNDS);
const isValid = await bcrypt.compare(password, passwordHash);
```

### Python

```python
from argon2 import PasswordHasher

ph = PasswordHasher(time_cost=3, memory_cost=65536, parallelism=4)

# Hash
password_hash = ph.hash(password)

# Verify
try:
    ph.verify(password_hash, password)
except argon2.exceptions.VerifyMismatchError:
    raise InvalidCredentials()
```

### Laravel

```php
// Laravel uses bcrypt by default -- just use the built-in
$hash = Hash::make($password);
$valid = Hash::check($password, $hash);

// Upgrade to Argon2id in config/hashing.php
'driver' => 'argon2id',
'memory' => 65536,
'threads' => 4,
'time' => 3,
```

## JWT Best Practices

### Token Configuration

```typescript
import { SignJWT, jwtVerify } from 'jose';

const SECRET = new TextEncoder().encode(env.JWT_SECRET); // Min 256 bits

// Sign
const token = await new SignJWT({ sub: userId, role: user.role })
  .setProtectedHeader({ alg: 'HS256' })
  .setIssuedAt()
  .setExpirationTime('15m')  // Short-lived access tokens
  .setAudience('https://myapp.com')
  .setIssuer('https://myapp.com')
  .sign(SECRET);

// Verify (check ALL claims)
const { payload } = await jwtVerify(token, SECRET, {
  audience: 'https://myapp.com',
  issuer: 'https://myapp.com',
  algorithms: ['HS256'],  // Prevent algorithm confusion
});
```

### JWT Rules

- Access tokens: 15 minutes max
- Refresh tokens: 7 days max, stored in httpOnly cookie
- NEVER store JWTs in localStorage (XSS can steal them)
- ALWAYS validate `exp`, `aud`, `iss`, and `alg`
- NEVER use `alg: "none"` -- always specify allowed algorithms
- Rotate signing keys periodically

## Brute-Force Protection

```typescript
import { RateLimiterMemory } from 'rate-limiter-flexible';

const loginLimiter = new RateLimiterMemory({
  points: 5,        // 5 attempts
  duration: 900,    // per 15 minutes
  blockDuration: 900, // block for 15 min after exceeded
});

app.post('/login', async (req, res) => {
  const key = `${req.ip}_${req.body.email}`;

  try {
    await loginLimiter.consume(key);
  } catch {
    return res.status(429).json({
      error: 'Too many login attempts. Try again in 15 minutes.',
    });
  }

  const user = await authenticate(req.body.email, req.body.password);

  if (!user) {
    // Don't reveal whether email exists
    return res.status(401).json({ error: 'Invalid email or password' });
  }

  // Reset limiter on success
  await loginLimiter.delete(key);
  return res.json({ token: generateToken(user) });
});
```

## Session Management

```typescript
// Secure cookie settings
const SESSION_OPTIONS = {
  httpOnly: true,      // JavaScript cannot access
  secure: true,        // HTTPS only
  sameSite: 'lax',     // CSRF protection
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
  path: '/',
  domain: '.myapp.com',
};

// Session fixation prevention: regenerate session ID on login
app.post('/login', async (req, res) => {
  const user = await authenticate(req.body);
  if (!user) return res.status(401).json({ error: 'Invalid credentials' });

  // Destroy old session and create new one
  req.session.regenerate((err) => {
    req.session.userId = user.id;
    req.session.save();
    res.json({ success: true });
  });
});

// Invalidate all sessions on password change
app.post('/change-password', async (req, res) => {
  await updatePassword(req.user.id, req.body.newPassword);
  await destroyAllSessions(req.user.id); // Force re-login everywhere
  res.json({ success: true, message: 'Please log in again' });
});
```

## Auth Checklist

- [ ] Passwords hashed with Argon2id or bcrypt (cost 12+)
- [ ] Login rate-limited (5 attempts per 15 min)
- [ ] Generic error messages ("Invalid email or password")
- [ ] Session regenerated on login (prevent fixation)
- [ ] All sessions invalidated on password change
- [ ] JWTs expire in 15 minutes or less
- [ ] Refresh tokens in httpOnly secure cookies
- [ ] No sensitive data in JWT payload
- [ ] Algorithm explicitly specified in JWT verification
- [ ] Account lockout after repeated failures

## PDPA 2024 Consequence

**Seksyen 5(1) dan 5(2) PDPA 2010 (Pindaan 2024):**
- Denda sehingga RM1,000,000
- Penjara sehingga 3 tahun

Weak authentication that allows unauthorized access to personal data is **kecuaian keselamatan** (security negligence). Using MD5/SHA-256 without proper stretching, or having no brute-force protection, demonstrates failure to implement reasonable security measures.
