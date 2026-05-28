---
name: security-headers
description: "Add security headers (CSP, HSTS, X-Frame-Options, Permissions-Policy) to prevent clickjacking, XSS, and data leaks. Provides copy-paste middleware for Express, Next.js, Cloudflare Workers, and Laravel. Use when hardening HTTP responses, fixing security scanner findings, or setting up a new project."
version: "1.0"
---

# Security Headers

Add HTTP security headers to every response. These are zero-effort, high-impact protections that prevent entire classes of attacks.

## The Rule

**EVERY HTTP response must include security headers. No exceptions.**

## Required Headers

| Header | What it prevents | Value |
|--------|-----------------|-------|
| `Strict-Transport-Security` | Downgrade attacks, SSL stripping | `max-age=31536000; includeSubDomains; preload` |
| `Content-Security-Policy` | XSS, data injection, clickjacking | See CSP section below |
| `X-Content-Type-Options` | MIME sniffing attacks | `nosniff` |
| `X-Frame-Options` | Clickjacking | `DENY` or `SAMEORIGIN` |
| `Referrer-Policy` | Leaking URLs to third parties | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | Unauthorized access to device APIs | `camera=(), microphone=(), geolocation=()` |
| `X-XSS-Protection` | Legacy XSS filter (older browsers) | `0` (disabled -- CSP is better) |

## Framework Implementations

### Cloudflare Workers

```typescript
function addSecurityHeaders(response: Response): Response {
  const headers = new Headers(response.headers);

  headers.set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains; preload');
  headers.set('X-Content-Type-Options', 'nosniff');
  headers.set('X-Frame-Options', 'DENY');
  headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
  headers.set('Permissions-Policy', 'camera=(), microphone=(), geolocation=()');
  headers.set('Content-Security-Policy', "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self' https:; frame-ancestors 'none'");

  // Remove headers that leak server info
  headers.delete('Server');
  headers.delete('X-Powered-By');

  return new Response(response.body, {
    status: response.status,
    statusText: response.statusText,
    headers,
  });
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const response = await handleRequest(request, env);
    return addSecurityHeaders(response);
  }
};
```

### Express.js / Node.js

```typescript
import helmet from 'helmet';

// Option 1: Use helmet (recommended)
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https:"],
      frameAncestors: ["'none'"],
    },
  },
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
}));

// Option 2: Manual middleware (no dependency)
app.use((req, res, next) => {
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains; preload');
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
  res.setHeader('Permissions-Policy', 'camera=(), microphone=(), geolocation=()');
  res.removeHeader('X-Powered-By');
  next();
});
```

### Next.js

```javascript
// next.config.js
const securityHeaders = [
  { key: 'Strict-Transport-Security', value: 'max-age=31536000; includeSubDomains; preload' },
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
];

module.exports = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: securityHeaders,
      },
    ];
  },
};
```

### Laravel / PHP

```php
// app/Http/Middleware/SecurityHeaders.php
class SecurityHeaders
{
    public function handle($request, Closure $next)
    {
        $response = $next($request);

        $response->headers->set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains; preload');
        $response->headers->set('X-Content-Type-Options', 'nosniff');
        $response->headers->set('X-Frame-Options', 'DENY');
        $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
        $response->headers->set('Permissions-Policy', 'camera=(), microphone=(), geolocation=()');
        $response->headers->remove('X-Powered-By');
        $response->headers->remove('Server');

        return $response;
    }
}

// Register in app/Http/Kernel.php
protected $middleware = [
    \App\Http\Middleware\SecurityHeaders::class,
    // ...
];
```

## Content-Security-Policy (CSP) Guide

CSP is the most complex header. Start strict, loosen only when needed:

### Strict (API-only, no frontend)

```
Content-Security-Policy: default-src 'none'; frame-ancestors 'none'
```

### Standard (SPA with CDN assets)

```
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self' https://api.yourapp.com; font-src 'self' https://fonts.gstatic.com; frame-ancestors 'none'
```

### With Third-Party Scripts (analytics, payments)

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://js.stripe.com https://www.googletagmanager.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self' https://api.stripe.com https://www.google-analytics.com; frame-src https://js.stripe.com; frame-ancestors 'none'
```

### CSP Report-Only (test before enforcing)

```
Content-Security-Policy-Report-Only: default-src 'self'; report-uri /csp-report
```

## Headers to Remove

These headers leak information about your stack:

```
Server: nginx/1.21.0          → Remove
X-Powered-By: Express         → Remove
X-AspNet-Version: 4.0.30319   → Remove
X-AspNetMvc-Version: 5.2      → Remove
```

## Verification

Test your headers at:
- Run `curl -I https://yoursite.com` and check response headers
- Use browser DevTools → Network tab → Response Headers

## PDPA 2024 Relevance

Security headers are part of **langkah keselamatan yang munasabah** (reasonable security measures) under PDPA 2024. Missing headers that lead to XSS or clickjacking attacks resulting in data breaches constitute negligence under Seksyen 5(1).
