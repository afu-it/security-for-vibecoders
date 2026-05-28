---
name: rate-limiting
description: "Add per-user and per-IP rate limiting to API endpoints. Covers Cloudflare Workers (KV-based sliding window), Express.js, Next.js, and generic patterns. Prevents abuse, brute-force attacks, and resource exhaustion. Use when adding rate limits, preventing API abuse, or fixing OWASP A04/A07 findings."
version: "1.0"
---

# Rate Limiting

Add per-user rate limiting to prevent abuse from compromised accounts, bots, or malicious actors. Addresses OWASP A04 (Insecure Design) and A07 (Authentication Failures).

## Why Rate Limiting Matters

Without rate limiting:
- A stolen session token can drain all credits in seconds
- A bot can create thousands of payment invoices
- A single user can DoS your service
- Brute-force attacks against auth endpoints succeed

## Cloudflare Workers (KV-based Sliding Window)

### Rate Limit Utility

```typescript
// lib/rate-limit.ts
import { auditLog } from './audit';

export type RateLimitResult =
  | { allowed: true }
  | { allowed: false; retryAfterSeconds: number };

export async function checkRateLimit(
  kv: KVNamespace,
  userId: string,
  endpoint: string,
  maxRequests: number,
  windowSeconds = 60,
): Promise<RateLimitResult> {
  const window = Math.floor(Date.now() / (windowSeconds * 1000));
  const key = `ratelimit:${userId}:${endpoint}:${window}`;

  const current = await kv.get(key);
  const count = current ? parseInt(current, 10) : 0;

  if (count >= maxRequests) {
    auditLog('auth.unauthorized', {
      reason: 'rate_limit_exceeded',
      userId,
      endpoint,
      count,
      maxRequests,
    });
    return { allowed: false, retryAfterSeconds: windowSeconds };
  }

  await kv.put(key, String(count + 1), { expirationTtl: windowSeconds * 2 });
  return { allowed: true };
}
```

### Usage in API Route

```typescript
import { checkRateLimit } from '@/lib/rate-limit';

export async function POST() {
  const { userId } = await auth();
  if (!userId) return new Response('Unauthorized', { status: 401 });

  const { PALANGIC_GUEST_KV } = await getBindings();

  // Max 10 requests per minute
  const rateCheck = await checkRateLimit(PALANGIC_GUEST_KV, userId, 'credits-spend', 10, 60);
  if (!rateCheck.allowed) {
    return Response.json(
      { error: 'rate_limit_exceeded', retryAfter: rateCheck.retryAfterSeconds },
      { status: 429, headers: { 'Retry-After': String(rateCheck.retryAfterSeconds) } },
    );
  }

  // ... proceed with operation
}
```

## Recommended Limits

| Endpoint | Limit | Window | Rationale |
|----------|-------|--------|-----------|
| Credit spend / export | 10/min | 60s | Normal user exports 1-3 at a time |
| Payment creation | 5/min | 60s | No legitimate reason to create 5+ invoices/min |
| Login attempts | 5/min | 60s | Brute-force prevention |
| Password reset | 3/hour | 3600s | Prevent email bombing |
| API general | 60/min | 60s | General abuse prevention |
| Guest/anonymous | 1/day | 86400s | Free tier limit |

## Express.js / Node.js

### Using express-rate-limit

```typescript
import rateLimit from 'express-rate-limit';

const apiLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 10,
  standardHeaders: true,
  legacyHeaders: false,
  keyGenerator: (req) => req.user?.id || req.ip,
  handler: (req, res) => {
    res.status(429).json({
      error: 'rate_limit_exceeded',
      retryAfter: 60,
    });
  },
});

app.use('/api/credits', apiLimiter);
```

### Using Redis (production)

```typescript
import { RateLimiterRedis } from 'rate-limiter-flexible';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

const rateLimiter = new RateLimiterRedis({
  storeClient: redis,
  keyPrefix: 'ratelimit',
  points: 10,       // max requests
  duration: 60,     // per 60 seconds
  blockDuration: 60, // block for 60s after exceeding
});

async function rateLimitMiddleware(req, res, next) {
  try {
    const key = req.user?.id || req.ip;
    await rateLimiter.consume(key);
    next();
  } catch (rejRes) {
    res.set('Retry-After', String(Math.ceil(rejRes.msBeforeNext / 1000)));
    res.status(429).json({ error: 'rate_limit_exceeded' });
  }
}
```

## Next.js (App Router)

### Middleware-based Rate Limiting

```typescript
// middleware.ts
import { NextResponse } from 'next/server';

const rateMap = new Map<string, { count: number; resetAt: number }>();

export function middleware(request: Request) {
  // Only rate-limit API routes
  if (!request.url.includes('/api/')) return NextResponse.next();

  const ip = request.headers.get('x-forwarded-for') || 'unknown';
  const now = Date.now();
  const window = 60_000; // 1 minute
  const maxRequests = 60;

  const entry = rateMap.get(ip);
  if (!entry || now > entry.resetAt) {
    rateMap.set(ip, { count: 1, resetAt: now + window });
    return NextResponse.next();
  }

  if (entry.count >= maxRequests) {
    return NextResponse.json(
      { error: 'rate_limit_exceeded' },
      { status: 429, headers: { 'Retry-After': '60' } },
    );
  }

  entry.count++;
  return NextResponse.next();
}
```

> **Note:** In-memory rate limiting only works for single-instance deployments. For serverless/edge, use KV, Redis, or Cloudflare Rate Limiting rules.

## Response Headers

Always include standard rate limit headers:

```typescript
const headers = {
  'Retry-After': '60',
  'X-RateLimit-Limit': '10',
  'X-RateLimit-Remaining': String(maxRequests - count),
  'X-RateLimit-Reset': String(Math.ceil(resetAt / 1000)),
};
```

## Testing Rate Limits

```typescript
test('rate limit blocks after max requests', async () => {
  // Make maxRequests calls
  for (let i = 0; i < 10; i++) {
    const res = await POST(makeRequest());
    expect(res.status).toBe(200);
  }

  // Next call should be blocked
  const blocked = await POST(makeRequest());
  expect(blocked.status).toBe(429);
  const body = await blocked.json();
  expect(body.error).toBe('rate_limit_exceeded');
  expect(body.retryAfter).toBeDefined();
});
```

## Cloudflare Rate Limiting Rules (Dashboard)

For IP-based rate limiting without code changes:

1. Cloudflare Dashboard → Security → WAF → Rate limiting rules
2. Create rule:
   - Match: `URI Path contains /api/`
   - Rate: 100 requests per 1 minute
   - Per: IP address
   - Action: Block for 60 seconds

This provides a baseline defense even if application-level rate limiting has bugs.

## PDPA 2024 Relevance

Rate limiting demonstrates:
- **Proportionate security measures** -- preventing abuse shows due diligence
- **Data minimization** -- limiting operations reduces exposure window
- **Incident prevention** -- stops automated attacks before they cause data breaches

A system without rate limiting that gets brute-forced is evidence of negligence under PDPA 2024.
