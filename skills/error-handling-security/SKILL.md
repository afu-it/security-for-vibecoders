---
name: error-handling-security
description: "Prevent information leakage through error messages. Covers sanitized error responses, structured error handling, no stack traces in production, and safe logging patterns for Express, Next.js, Cloudflare Workers, Laravel, and FastAPI. Use when fixing verbose errors, implementing error handlers, or preventing stack trace leaks."
version: "1.0"
---

# Error Handling Security

Prevent stack traces, database errors, and internal details from leaking to attackers. Verbose errors in production are a reconnaissance goldmine -- they reveal your framework, database structure, file paths, and dependencies.

## The Rule

**NEVER expose internal error details to users. ALWAYS return generic messages externally and log details internally.**

## What Leaks Look Like

### BAD -- Exposes everything to attacker:

```json
{
  "error": "ER_NO_SUCH_TABLE: Table 'myapp_prod.users_v2' doesn't exist",
  "stack": "Error: ER_NO_SUCH_TABLE\n    at /app/src/services/user.ts:47:12\n    at Pool.query (/app/node_modules/mysql2/promise.js:94:22)",
  "sql": "SELECT * FROM users_v2 WHERE email = 'admin@test.com'"
}
```

This tells the attacker: database type (MySQL), table names, file structure, dependencies, and the exact query being run.

### GOOD -- Generic external, detailed internal:

```json
{
  "error": "Something went wrong",
  "code": "INTERNAL_ERROR",
  "requestId": "req_abc123"
}
```

Meanwhile, the full error is logged server-side with the requestId for debugging.

## Framework Implementations

### Cloudflare Workers

```typescript
interface AppError {
  code: string;
  message: string;       // Safe for users
  internal?: string;     // Never sent to client
  status: number;
}

function handleError(error: unknown, requestId: string): Response {
  // Log full error internally
  console.error(JSON.stringify({
    requestId,
    error: error instanceof Error ? error.message : String(error),
    stack: error instanceof Error ? error.stack : undefined,
    timestamp: new Date().toISOString(),
  }));

  // Determine safe response
  if (error instanceof AppError) {
    return Response.json(
      { error: error.message, code: error.code, requestId },
      { status: error.status }
    );
  }

  // Unknown errors -- NEVER expose details
  return Response.json(
    { error: 'Internal server error', code: 'INTERNAL_ERROR', requestId },
    { status: 500 }
  );
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const requestId = crypto.randomUUID();
    try {
      return await handleRequest(request, env);
    } catch (error) {
      return handleError(error, requestId);
    }
  }
};
```

### Express.js

BAD:
```typescript
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message, stack: err.stack });
});
```

GOOD:
```typescript
// Custom error class for expected errors
class AppError extends Error {
  constructor(
    public statusCode: number,
    public code: string,
    message: string,        // Safe for users
    public internal?: string // For logs only
  ) {
    super(message);
  }
}

// Global error handler -- LAST middleware
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  const requestId = req.headers['x-request-id'] || crypto.randomUUID();

  // Log full error internally
  console.error(JSON.stringify({
    requestId,
    method: req.method,
    path: req.path,
    error: err.message,
    stack: err.stack,
    timestamp: new Date().toISOString(),
  }));

  // Send safe response
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      error: err.message,
      code: err.code,
      requestId,
    });
  }

  // Unknown errors -- generic message only
  res.status(500).json({
    error: 'Internal server error',
    code: 'INTERNAL_ERROR',
    requestId,
  });
});

// Usage in routes
app.get('/users/:id', async (req, res, next) => {
  try {
    const user = await getUser(req.params.id);
    if (!user) throw new AppError(404, 'NOT_FOUND', 'User not found');
    res.json(user);
  } catch (err) {
    next(err); // Forward to error handler
  }
});
```

### Next.js

```typescript
// app/api/users/route.ts
export async function GET(request: Request) {
  try {
    const users = await getUsers();
    return Response.json(users);
  } catch (error) {
    console.error('[API /users]', error);

    // Never expose error details in production
    return Response.json(
      { error: 'Failed to fetch users', code: 'FETCH_ERROR' },
      { status: 500 }
    );
  }
}
```

### Laravel

```php
// app/Exceptions/Handler.php
class Handler extends ExceptionHandler
{
    // Never report these to error tracking
    protected $dontReport = [
        AuthenticationException::class,
        ValidationException::class,
    ];

    public function render($request, Throwable $e)
    {
        if ($request->expectsJson()) {
            $requestId = $request->header('X-Request-ID', Str::uuid());

            // Log full error
            Log::error('API Error', [
                'request_id' => $requestId,
                'exception' => $e->getMessage(),
                'trace' => $e->getTraceAsString(),
                'url' => $request->fullUrl(),
            ]);

            // Known errors
            if ($e instanceof ModelNotFoundException) {
                return response()->json([
                    'error' => 'Resource not found',
                    'code' => 'NOT_FOUND',
                    'request_id' => $requestId,
                ], 404);
            }

            // Unknown errors -- generic only
            return response()->json([
                'error' => 'Internal server error',
                'code' => 'INTERNAL_ERROR',
                'request_id' => $requestId,
            ], 500);
        }

        return parent::render($request, $e);
    }
}

// Ensure APP_DEBUG=false in production .env
```

### Python FastAPI

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import logging, uuid

app = FastAPI()
logger = logging.getLogger(__name__)

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    request_id = str(uuid.uuid4())

    # Log full error internally
    logger.error(f"[{request_id}] {request.method} {request.url} - {exc}", exc_info=True)

    # Generic response externally
    return JSONResponse(
        status_code=500,
        content={
            "error": "Internal server error",
            "code": "INTERNAL_ERROR",
            "request_id": request_id,
        },
    )
```

## Environment Configuration

```bash
# .env.production
NODE_ENV=production
APP_DEBUG=false
LOG_LEVEL=error

# NEVER in production:
# DEBUG=*
# APP_DEBUG=true
# SHOW_ERRORS=true
```

## Error Handling Checklist

- [ ] No stack traces in API responses (production)
- [ ] No SQL queries in error messages
- [ ] No file paths in error messages
- [ ] No dependency versions in error messages
- [ ] `APP_DEBUG=false` / `NODE_ENV=production` in production
- [ ] Generic error messages for unknown errors
- [ ] Request ID in every error response (for support debugging)
- [ ] Full errors logged server-side with request ID correlation
- [ ] Validation errors return field-specific messages (safe)
- [ ] 404 errors don't reveal whether a resource exists for unauthorized users

## PDPA 2024 Relevance

Verbose error messages that reveal database structure, user data, or system internals constitute an information disclosure vulnerability. Under PDPA 2024, this aids attackers in exploiting the system to access personal data -- making the developer liable for failing to implement reasonable security measures under Seksyen 5(1).
