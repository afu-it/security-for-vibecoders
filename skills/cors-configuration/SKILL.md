---
name: cors-configuration
description: "Configure CORS correctly to prevent credential leaks and unauthorized cross-origin access. Covers Express, Next.js, Cloudflare Workers, Laravel, and FastAPI. Use when setting up CORS, fixing preflight errors, or auditing cross-origin policies."
version: "1.0"
---

# CORS Configuration

Configure Cross-Origin Resource Sharing correctly. Misconfigured CORS is one of the most common vibe-coder mistakes -- wildcard origins with credentials enabled is equivalent to having no security at all.

## The Rule

**NEVER use `Access-Control-Allow-Origin: *` with credentials. ALWAYS whitelist specific origins.**

## Common Mistakes

| Mistake | Risk | Fix |
|---------|------|-----|
| `Origin: *` with credentials | Any site can steal user data | Whitelist specific origins |
| Reflecting `Origin` header without validation | Attacker controls allowed origin | Check against allowlist |
| Allowing `null` origin | Sandboxed iframes can exploit | Never allow `null` |
| Overly broad regex (`*.example.com`) | `evil-example.com` matches | Use exact string matching |
| Exposing all headers | Information leakage | Only expose needed headers |

## Framework Implementations

### Cloudflare Workers

```typescript
const ALLOWED_ORIGINS = [
  'https://myapp.com',
  'https://www.myapp.com',
  'https://staging.myapp.com',
];

// Add localhost in development only
if (env.ENVIRONMENT === 'development') {
  ALLOWED_ORIGINS.push('http://localhost:3000');
}

function corsHeaders(request: Request): HeadersInit {
  const origin = request.headers.get('Origin') || '';

  if (!ALLOWED_ORIGINS.includes(origin)) {
    return {}; // No CORS headers = browser blocks the request
  }

  return {
    'Access-Control-Allow-Origin': origin,
    'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    'Access-Control-Allow-Credentials': 'true',
    'Access-Control-Max-Age': '86400',
  };
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Handle preflight
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        status: 204,
        headers: corsHeaders(request),
      });
    }

    const response = await handleRequest(request, env);
    const headers = new Headers(response.headers);

    // Add CORS headers to actual response
    for (const [key, value] of Object.entries(corsHeaders(request))) {
      headers.set(key, value);
    }

    return new Response(response.body, {
      status: response.status,
      headers,
    });
  }
};
```

### Express.js

BAD:
```typescript
import cors from 'cors';
app.use(cors()); // Allows ALL origins -- dangerous!
```

GOOD:
```typescript
import cors from 'cors';

const ALLOWED_ORIGINS = [
  'https://myapp.com',
  'https://www.myapp.com',
];

app.use(cors({
  origin: (origin, callback) => {
    // Allow requests with no origin (mobile apps, curl)
    if (!origin) return callback(null, true);

    if (ALLOWED_ORIGINS.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400,
}));
```

### Next.js API Routes

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

const ALLOWED_ORIGINS = [
  'https://myapp.com',
  'https://www.myapp.com',
];

export function middleware(request: NextRequest) {
  const origin = request.headers.get('origin') || '';
  const isAllowed = ALLOWED_ORIGINS.includes(origin);

  // Handle preflight
  if (request.method === 'OPTIONS') {
    return new NextResponse(null, {
      status: 204,
      headers: {
        'Access-Control-Allow-Origin': isAllowed ? origin : '',
        'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
        'Access-Control-Allow-Headers': 'Content-Type, Authorization',
        'Access-Control-Allow-Credentials': 'true',
        'Access-Control-Max-Age': '86400',
      },
    });
  }

  const response = NextResponse.next();
  if (isAllowed) {
    response.headers.set('Access-Control-Allow-Origin', origin);
    response.headers.set('Access-Control-Allow-Credentials', 'true');
  }
  return response;
}

export const config = { matcher: '/api/:path*' };
```

### Laravel

```php
// config/cors.php
return [
    'paths' => ['api/*'],
    'allowed_methods' => ['GET', 'POST', 'PUT', 'DELETE'],
    'allowed_origins' => [
        'https://myapp.com',
        'https://www.myapp.com',
    ],
    'allowed_origins_patterns' => [], // Avoid regex patterns
    'allowed_headers' => ['Content-Type', 'Authorization'],
    'exposed_headers' => [],
    'max_age' => 86400,
    'supports_credentials' => true,
];
```

### Python FastAPI

```python
from fastapi.middleware.cors import CORSMiddleware

ALLOWED_ORIGINS = [
    "https://myapp.com",
    "https://www.myapp.com",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,  # Never use ["*"] with credentials
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Content-Type", "Authorization"],
    max_age=86400,
)
```

## CORS Checklist

- [ ] Origins are explicitly whitelisted (no wildcards with credentials)
- [ ] `Access-Control-Allow-Origin` never reflects untrusted input
- [ ] `null` origin is not in the allowlist
- [ ] Preflight (OPTIONS) requests are handled correctly
- [ ] `Access-Control-Max-Age` is set to reduce preflight requests
- [ ] Only necessary methods and headers are allowed
- [ ] Development origins (localhost) are excluded in production

## PDPA 2024 Relevance

Misconfigured CORS that allows any website to access authenticated API endpoints constitutes a failure to implement **langkah keselamatan yang munasabah** (reasonable security measures). If user data is exfiltrated through CORS misconfiguration, the developer faces liability under Seksyen 5(1).
