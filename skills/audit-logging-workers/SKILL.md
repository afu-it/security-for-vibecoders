---
name: audit-logging-workers
description: "Add structured audit logging with immutable R2 storage to Cloudflare Workers. Creates a typed audit utility, instruments all API routes, deploys a tail worker for persistent log storage, and configures R2 bucket lock for tamper-resistance. Use when adding logging, audit trails, or PDPA compliance to a Cloudflare Workers project."
version: "1.0"
---

# Audit Logging for Cloudflare Workers

Add forensic-grade structured audit logging to any Cloudflare Workers project. Satisfies PDPA 2024 requirement for traceable records of security-relevant events.

## Architecture

```
API Route → auditLog() → console.log(JSON)
                              ↓
                    Cloudflare Workers Runtime
                              ↓
                    Tail Worker (palangic-audit-tail)
                              ↓
                    R2 Bucket (NDJSON, partitioned by date/hour)
                              ↓
                    Object Lock (365-day compliance retention)
```

## Step 1: Create the Audit Utility

Create `lib/audit.ts`:

```typescript
export type AuditEvent =
  | 'webhook.rejected'
  | 'webhook.duplicate'
  | 'webhook.credited'
  | 'webhook.error'
  | 'credits.spent'
  | 'credits.insufficient'
  | 'auth.unauthorized'
  | 'payment.created'
  | 'payment.error'
  | 'guest.limit_reached'
  | 'guest.spent'
  | 'worker.exception';

export function generateRequestId(): string {
  return crypto.randomUUID();
}

export function auditLog(event: AuditEvent, data?: Record<string, unknown>, requestId?: string): void {
  const entry = {
    level: 'audit',
    event,
    ts: new Date().toISOString(),
    requestId: requestId ?? null,
    ...(data && { data }),
  };
  console.log(JSON.stringify(entry));
}
```

## Step 2: Instrument API Routes

Every API route should:
1. Generate a request ID at the start
2. Log security-relevant events with that ID
3. Include user context where available

```typescript
import { auditLog, generateRequestId } from '@/lib/audit';

export async function POST(request: Request) {
  const reqId = generateRequestId();

  // Log auth failures
  if (!userId) {
    auditLog('auth.unauthorized', { route: '/api/example' }, reqId);
    return new Response('Unauthorized', { status: 401 });
  }

  // Log successful operations
  auditLog('credits.spent', { userId, amount: -1 }, reqId);

  // Log errors
  auditLog('payment.error', { userId, error: 'timeout' }, reqId);
}
```

## Step 3: Create Tail Worker

Create `workers/audit-tail/src/index.ts`:

```typescript
export interface Env {
  AUDIT_LOGS_BUCKET: R2Bucket;
}

interface TailMessage {
  readonly scriptName: string | null;
  readonly event: { readonly request?: { readonly url: string; readonly method: string } } | null;
  readonly logs: { readonly level: string; readonly message: readonly string[]; readonly timestamp: number }[];
  readonly exceptions: { readonly name: string; readonly message: string; readonly timestamp: number }[];
}

export default {
  async tail(events: TailMessage[], env: Env): Promise<void> {
    const auditLines: string[] = [];

    for (const event of events) {
      for (const log of event.logs) {
        if (log.message.length === 0) continue;
        try {
          const parsed = JSON.parse(log.message[0]);
          if (parsed.level === 'audit') {
            auditLines.push(JSON.stringify({
              ...parsed,
              _worker: event.scriptName,
              _requestUrl: event.event?.request?.url ?? null,
              _requestMethod: event.event?.request?.method ?? null,
              _ingestedAt: new Date().toISOString(),
            }));
          }
        } catch { /* not audit entry */ }
      }

      for (const exception of event.exceptions) {
        auditLines.push(JSON.stringify({
          level: 'audit',
          event: 'worker.exception',
          ts: new Date(exception.timestamp).toISOString(),
          data: { name: exception.name, message: exception.message },
          _worker: event.scriptName,
          _ingestedAt: new Date().toISOString(),
        }));
      }
    }

    if (auditLines.length === 0) return;

    const now = new Date();
    const key = `audit-logs/${now.getUTCFullYear()}/${String(now.getUTCMonth() + 1).padStart(2, '0')}/${String(now.getUTCDate()).padStart(2, '0')}/${String(now.getUTCHours()).padStart(2, '0')}/batch-${now.getTime()}.ndjson`;

    await env.AUDIT_LOGS_BUCKET.put(key, auditLines.join('\n') + '\n', {
      httpMetadata: { contentType: 'application/x-ndjson' },
      customMetadata: { entries: String(auditLines.length) },
    });
  },
};
```

## Step 4: Configure Wrangler

Main worker `wrangler.jsonc` -- add:
```jsonc
{
  "r2_buckets": [{ "binding": "AUDIT_LOGS_BUCKET", "bucket_name": "your-audit-logs" }],
  "tail_consumers": [{ "service": "your-audit-tail" }]
}
```

Tail worker `wrangler.jsonc`:
```jsonc
{
  "name": "your-audit-tail",
  "main": "src/index.ts",
  "compatibility_date": "2025-04-14",
  "r2_buckets": [{ "binding": "AUDIT_LOGS_BUCKET", "bucket_name": "your-audit-logs" }]
}
```

## Step 5: Deploy and Lock

```bash
# Create R2 bucket
npx wrangler r2 bucket create your-audit-logs

# Deploy tail worker
cd workers/audit-tail && npx wrangler deploy

# Enable Object Lock via Cloudflare Dashboard:
# R2 → your-audit-logs → Settings → Bucket Lock Rules → Add Rule
# Type: Compliance, Duration: 365 days
```

## What Gets Logged

| Event | When | Data |
|-------|------|------|
| `auth.unauthorized` | Unauthenticated request to protected route | route, requestId |
| `webhook.rejected` | Invalid webhook token or payload | reason, error |
| `webhook.duplicate` | Replayed webhook (idempotency caught) | externalId, userId |
| `webhook.credited` | Successful payment credit | externalId, userId, credits |
| `credits.spent` | User exported a watermarked file | userId, amount |
| `credits.insufficient` | User tried to export with 0 credits | userId |
| `payment.created` | Invoice created successfully | userId, packKey, invoiceId |
| `payment.error` | Xendit API failure | userId, status, error |
| `guest.limit_reached` | Guest hit daily limit | fingerprint |
| `worker.exception` | Unhandled exception in worker | name, message |

## PDPA 2024 Compliance

This satisfies:
- **Seksyen 5(1):** Traceable record of all data processing activities
- **Seksyen 5(2):** Security controls proportionate to data sensitivity
- **Forensic requirement:** Immutable logs that cannot be altered after write (R2 Object Lock)
- **Incident response:** Request IDs enable correlating events across a single request flow

## Important Notes

- Audit logs do NOT contain image data, personal documents, or payment credentials
- Request IDs are UUIDs with no PII
- R2 Object Lock in Compliance mode means NO ONE can delete logs (not even account owner)
- Partition by date/hour enables efficient querying for incident response
