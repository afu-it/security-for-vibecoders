---
name: webhook-security
description: "Fix race conditions, add idempotency guards, and implement timing-safe token comparisons for payment webhooks. Covers Xendit, Stripe, Paddle, and generic webhook patterns. Prevents double-crediting, replay attacks, and timing attacks. Use when building payment webhooks, fixing race conditions, or securing callback endpoints."
version: "1.0"
---

# Webhook Security

Secure payment webhook endpoints against the three most common vulnerabilities: race conditions (double-credit), replay attacks, and timing attacks.

## The Three Vulnerabilities

### 1. TOCTOU Race Condition (Double-Credit)

**The bug:** Check if webhook was already processed (SELECT), then process it (INSERT + UPDATE). Two concurrent webhooks both pass the SELECT before either commits.

BAD (vulnerable):
```typescript
// Check-then-act -- RACE CONDITION
const existing = await db.prepare('SELECT id FROM transactions WHERE ref = ?').bind(ref).first();
if (!existing) {
  await addCredits(db, userId, credits, ref); // Both concurrent requests reach here
}
```

GOOD (atomic idempotency):
```typescript
// INSERT OR IGNORE -- atomic, no race window
const result = await db.prepare(
  "INSERT OR IGNORE INTO transactions (user_id, amount, reason, ref, created_at) VALUES (?, ?, ?, ?, datetime('now'))"
).bind(userId, credits, 'purchase', ref).run();

if (result.meta.changes === 0) {
  // Duplicate -- already processed, return success without crediting
  return new Response(JSON.stringify({ ok: true }), { status: 200 });
}

// Only reaches here if INSERT succeeded (new transaction)
await db.prepare(
  "UPDATE user_credits SET credits = credits + ? WHERE user_id = ?"
).bind(credits, userId).run();
```

**Why this works:** The UNIQUE constraint on `ref` column makes the INSERT atomic. If two requests arrive simultaneously, only one INSERT succeeds (changes === 1). The other gets changes === 0 and returns early.

### 2. Timing Attack on Token Verification

**The bug:** Using `===` or `!==` for token comparison leaks information about the correct token through response timing.

BAD (vulnerable):
```typescript
if (callbackToken !== expectedToken) {
  return new Response('Forbidden', { status: 403 });
}
```

GOOD (timing-safe):
```typescript
function timingSafeEqual(a: string, b: string): boolean {
  const encoder = new TextEncoder();
  const bufA = encoder.encode(a);
  const bufB = encoder.encode(b);

  if (bufA.byteLength !== bufB.byteLength) return false;

  let mismatch = 0;
  for (let i = 0; i < bufA.byteLength; i++) {
    mismatch |= bufA[i] ^ bufB[i];
  }
  return mismatch === 0;
}

if (!expectedToken || !callbackToken || !timingSafeEqual(callbackToken, expectedToken)) {
  return new Response('Forbidden', { status: 403 });
}
```

**Why this works:** XOR comparison always iterates all bytes regardless of where the mismatch is. An attacker cannot determine correct bytes by measuring response time.

### 3. Replay Attack (Missing Idempotency)

**The bug:** Processing the same webhook multiple times because there's no deduplication.

**Fix:** Always store a unique reference (external_id, invoice_id) with a UNIQUE database constraint. The INSERT OR IGNORE pattern from fix #1 handles this automatically.

## Complete Webhook Handler Template

```typescript
import { auditLog, generateRequestId } from '@/lib/audit';

export async function POST(request: Request) {
  const reqId = generateRequestId();

  // 1. Verify token (timing-safe)
  const callbackToken = request.headers.get('x-callback-token');
  if (!verifyToken(callbackToken, env.WEBHOOK_TOKEN)) {
    auditLog('webhook.rejected', { reason: 'invalid_token' }, reqId);
    return new Response(JSON.stringify({ error: 'forbidden' }), { status: 403 });
  }

  // 2. Validate payload structure
  const body = await request.json().catch(() => null);
  if (!isValidPayload(body)) {
    auditLog('webhook.rejected', { reason: 'invalid_payload' }, reqId);
    return new Response(JSON.stringify({ error: 'invalid_payload' }), { status: 400 });
  }

  // 3. Skip non-paid events
  if (body.status !== 'PAID') {
    auditLog('webhook.skipped', { reason: 'not_paid' }, reqId);
    return new Response(JSON.stringify({ ok: true }), { status: 200 });
  }

  // 4. Validate credits match expected pack (defense-in-depth)
  const expectedCredits = PACKS[body.metadata.pack]?.credits;
  if (body.metadata.credits !== expectedCredits) {
    auditLog('webhook.rejected', { reason: 'credits_mismatch' }, reqId);
    return new Response(JSON.stringify({ error: 'invalid_payload' }), { status: 400 });
  }

  // 5. Atomic idempotent credit (INSERT OR IGNORE)
  try {
    const result = await db.prepare(
      "INSERT OR IGNORE INTO transactions (user_id, amount, reason, ref, created_at) VALUES (?, ?, ?, ?, datetime('now'))"
    ).bind(userId, credits, 'purchase', externalId).run();

    if (result.meta.changes === 0) {
      auditLog('webhook.duplicate', { externalId, userId }, reqId);
      return new Response(JSON.stringify({ ok: true }), { status: 200 });
    }

    await db.prepare(
      "UPDATE user_credits SET credits = credits + ?, updated_at = datetime('now') WHERE user_id = ?"
    ).bind(credits, userId).run();

    auditLog('webhook.credited', { externalId, userId, credits }, reqId);
  } catch (err) {
    auditLog('webhook.error', { externalId, error: String(err) }, reqId);
    return new Response(JSON.stringify({ error: 'internal_error' }), { status: 500 });
  }

  return new Response(JSON.stringify({ ok: true }), { status: 200 });
}
```

## Database Schema Requirements

```sql
CREATE TABLE credit_transactions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id TEXT NOT NULL,
  amount INTEGER NOT NULL,
  reason TEXT NOT NULL,
  xendit_ref TEXT,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  UNIQUE(xendit_ref)  -- THIS IS CRITICAL for idempotency
);

CREATE INDEX idx_transactions_user ON credit_transactions(user_id);
CREATE UNIQUE INDEX idx_transactions_ref ON credit_transactions(xendit_ref);
```

## Provider-Specific Notes

### Xendit
- Token header: `x-callback-token`
- Static shared secret (not HMAC)
- Retries on 5xx responses

### Stripe
- Token header: `stripe-signature`
- HMAC-SHA256 signature over request body
- Use `stripe.webhooks.constructEvent()` for verification

### Paddle
- Token header: `paddle-signature`
- HMAC-SHA256 with timestamp
- Use Paddle SDK verification

## Testing Webhook Security

```typescript
test('replayed webhook does not double-credit', async () => {
  // First call should credit
  const res1 = await POST(makeWebhookRequest('invoice_123'));
  expect(res1.status).toBe(200);
  expect(creditsAdded).toBe(1);

  // Second call with same ID should NOT credit
  const res2 = await POST(makeWebhookRequest('invoice_123'));
  expect(res2.status).toBe(200);
  expect(creditsAdded).toBe(1); // Still 1, not 2
});

test('invalid token returns 403', async () => {
  const res = await POST(makeWebhookRequest('invoice_123', { token: 'wrong' }));
  expect(res.status).toBe(403);
});
```

## PDPA 2024 Relevance

Race conditions in payment processing can lead to:
- Financial loss (double-crediting)
- Data integrity violations (inconsistent records)
- Inability to reconcile transactions during audit

Under PDPA 2024, a Data Processor must implement "appropriate security measures." A payment system with known race conditions fails this standard.
