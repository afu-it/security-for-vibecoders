---
name: ssrf-prevention
description: "Enterprise-grade Server-Side Request Forgery (SSRF) prevention for APIs, webhooks, URL previewers, importers, and fetch-by-URL features. Covers strict allowlists, DNS rebinding defense, private IP blocking, cloud metadata protection, redirect validation, egress controls, and safe fetch wrappers for Node.js, Cloudflare Workers, Python, and Laravel. Use when code fetches user-provided URLs or calls external services."
version: "1.0"
---

# SSRF Prevention

Prevent attackers from using your server as a proxy into internal networks, cloud metadata endpoints, databases, admin panels, and private services. SSRF is OWASP A10 and is commonly introduced by vibe-coded features such as URL previews, webhook testers, image importers, PDF generators, and "fetch this URL" integrations.

## The Rule

**NEVER fetch a user-provided URL directly. ALWAYS validate destination, resolve DNS safely, block private networks, and enforce egress allowlists.**

## What SSRF Can Expose

| Target | Why it matters |
|--------|----------------|
| `169.254.169.254` | AWS/GCP/Azure instance metadata credentials |
| `127.0.0.1` / `localhost` | Local admin dashboards, debug servers, Redis, Docker APIs |
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | Internal networks and private services |
| IPv6 loopback/link-local | `::1`, `fe80::/10`, IPv4-mapped bypasses |
| Internal DNS names | `db.internal`, `admin.local`, Kubernetes service names |
| Redirect chains | Valid public URL redirects to internal endpoint |

## Vulnerable Patterns

BAD:

```typescript
// URL previewer
app.post('/preview', async (req, res) => {
  const html = await fetch(req.body.url).then(r => r.text());
  res.json({ title: extractTitle(html) });
});

// Webhook tester
await fetch(user.webhookUrl, { method: 'POST', body: JSON.stringify(payload) });
```

GOOD:

```typescript
const html = await safeFetchUserUrl(req.body.url, {
  allowedProtocols: ['https:'],
  allowedHosts: ['example.com', 'api.partner.com'],
  maxRedirects: 0,
  timeoutMs: 3000,
});
```

## Enterprise Control Model

Use layered controls. Do not rely on only one check.

1. **Positive allowlist** -- approved hostnames or domains only
2. **Protocol restriction** -- HTTPS only, no `file:`, `gopher:`, `ftp:`, `dict:`
3. **DNS resolution check** -- resolve hostname and block private/reserved IPs
4. **Redirect validation** -- every redirect target must pass the same checks
5. **Timeout and size limits** -- prevent resource exhaustion
6. **Egress firewall** -- network-level deny to metadata/internal ranges
7. **Audit logging** -- log every blocked SSRF attempt with user ID and URL

## Blocked IP Ranges

Block all private, loopback, link-local, multicast, reserved, and cloud metadata ranges:

```text
0.0.0.0/8
10.0.0.0/8
100.64.0.0/10
127.0.0.0/8
169.254.0.0/16
172.16.0.0/12
192.0.0.0/24
192.0.2.0/24
192.168.0.0/16
198.18.0.0/15
198.51.100.0/24
203.0.113.0/24
224.0.0.0/4
240.0.0.0/4
::/128
::1/128
::ffff:0:0/96
fc00::/7
fe80::/10
ff00::/8
```

Special cloud metadata endpoints:

```text
169.254.169.254
metadata.google.internal
metadata.azure.com
100.100.100.200
```

## Node.js Safe Fetch Wrapper

```typescript
import dns from 'node:dns/promises';
import ipaddr from 'ipaddr.js';

const ALLOWED_HOSTS = new Set([
  'api.stripe.com',
  'api.xendit.co',
  'example.com',
]);

function isBlockedIp(address: string): boolean {
  const parsed = ipaddr.parse(address);
  const range = parsed.range();
  return [
    'unspecified',
    'broadcast',
    'multicast',
    'linkLocal',
    'loopback',
    'private',
    'reserved',
    'uniqueLocal',
  ].includes(range);
}

async function assertSafeDestination(url: URL): Promise<void> {
  if (url.protocol !== 'https:') {
    throw new Error('Only HTTPS URLs are allowed');
  }

  if (!ALLOWED_HOSTS.has(url.hostname)) {
    throw new Error('Host is not allowlisted');
  }

  const records = await dns.lookup(url.hostname, { all: true, verbatim: true });
  if (records.length === 0) {
    throw new Error('Host did not resolve');
  }

  for (const record of records) {
    if (isBlockedIp(record.address)) {
      throw new Error('Resolved IP is blocked');
    }
  }
}

export async function safeFetchUserUrl(input: string): Promise<Response> {
  const url = new URL(input);
  await assertSafeDestination(url);

  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), 3000);

  try {
    const response = await fetch(url, {
      redirect: 'manual',
      signal: controller.signal,
      headers: {
        'User-Agent': 'secure-ship-fetch/1.0',
      },
    });

    if (response.status >= 300 && response.status < 400) {
      throw new Error('Redirects are disabled for user-provided URLs');
    }

    const length = response.headers.get('content-length');
    if (length && Number(length) > 1_000_000) {
      throw new Error('Response too large');
    }

    return response;
  } finally {
    clearTimeout(timeout);
  }
}
```

## Cloudflare Workers Pattern

Cloudflare Workers cannot perform raw DNS lookups from user code. Prefer strict hostname allowlists and Cloudflare-level egress controls.

```typescript
const ALLOWED_HOSTS = new Set([
  'api.stripe.com',
  'api.xendit.co',
]);

function validateOutboundUrl(input: string): URL {
  const url = new URL(input);

  if (url.protocol !== 'https:') {
    throw new Error('Only HTTPS is allowed');
  }

  if (!ALLOWED_HOSTS.has(url.hostname)) {
    throw new Error('Host is not allowlisted');
  }

  if (url.username || url.password) {
    throw new Error('URL credentials are not allowed');
  }

  return url;
}

export async function safeOutboundFetch(input: string, init?: RequestInit): Promise<Response> {
  const url = validateOutboundUrl(input);

  return fetch(url.toString(), {
    ...init,
    redirect: 'manual',
    signal: AbortSignal.timeout(3000),
  });
}
```

## Python Pattern

```python
import ipaddress
import socket
import requests
from urllib.parse import urlparse

ALLOWED_HOSTS = {"api.stripe.com", "api.xendit.co", "example.com"}

def is_blocked_ip(address: str) -> bool:
    ip = ipaddress.ip_address(address)
    return (
        ip.is_private
        or ip.is_loopback
        or ip.is_link_local
        or ip.is_multicast
        or ip.is_reserved
        or ip.is_unspecified
    )

def assert_safe_url(raw_url: str):
    parsed = urlparse(raw_url)

    if parsed.scheme != "https":
        raise ValueError("Only HTTPS URLs are allowed")

    if parsed.hostname not in ALLOWED_HOSTS:
        raise ValueError("Host is not allowlisted")

    for result in socket.getaddrinfo(parsed.hostname, None):
        address = result[4][0]
        if is_blocked_ip(address):
            raise ValueError("Resolved IP is blocked")

def safe_fetch(raw_url: str):
    assert_safe_url(raw_url)
    response = requests.get(
        raw_url,
        timeout=3,
        allow_redirects=False,
        headers={"User-Agent": "secure-ship-fetch/1.0"},
    )

    if 300 <= response.status_code < 400:
        raise ValueError("Redirects are disabled")

    if int(response.headers.get("content-length", "0")) > 1_000_000:
        raise ValueError("Response too large")

    return response
```

## Laravel Pattern

```php
use Illuminate\Support\Facades\Http;

function assertSafeUrl(string $rawUrl): string
{
    $parts = parse_url($rawUrl);

    if (($parts['scheme'] ?? '') !== 'https') {
        throw new InvalidArgumentException('Only HTTPS URLs are allowed');
    }

    $allowedHosts = ['api.stripe.com', 'api.xendit.co', 'example.com'];
    $host = $parts['host'] ?? '';

    if (!in_array($host, $allowedHosts, true)) {
        throw new InvalidArgumentException('Host is not allowlisted');
    }

    $ips = gethostbynamel($host) ?: [];
    foreach ($ips as $ip) {
        if (!filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE)) {
            throw new InvalidArgumentException('Resolved IP is blocked');
        }
    }

    return $rawUrl;
}

$response = Http::timeout(3)
    ->withoutRedirecting()
    ->get(assertSafeUrl($url));
```

## DNS Rebinding Defense

DNS rebinding happens when a hostname first resolves to a public IP during validation, then resolves to a private IP during the actual request.

Required defenses:

- Resolve the hostname yourself before the request
- Use a custom HTTP agent that connects to the validated IP while preserving the Host header (Node.js advanced case)
- Disable redirects or revalidate every redirect target
- Prefer strict allowlisted partner hostnames over arbitrary user URLs
- Add network-level egress blocks to private and metadata IP ranges

## Redirect Rules

Redirects are a common bypass.

BAD:

```typescript
await fetch(url); // follows redirects by default
```

GOOD:

```typescript
await fetch(url, { redirect: 'manual' });
```

If redirects are required, validate every `Location` URL with the same SSRF checks before following it.

## Egress Firewall Requirements

Application checks are not enough. Production systems should also block outbound traffic to internal ranges at the network layer.

Minimum egress policy:

- Deny all outbound traffic by default
- Allow only required third-party APIs
- Deny cloud metadata IPs explicitly
- Deny RFC1918 private networks
- Log denied egress attempts

## Audit Checklist

- [ ] No raw `fetch(userUrl)`, `requests.get(user_url)`, or `Http::get($userUrl)`
- [ ] Outbound requests use a safe wrapper
- [ ] HTTPS-only enforced
- [ ] Positive hostname allowlist used where possible
- [ ] Private, loopback, link-local, reserved, and metadata IPs blocked
- [ ] Redirects disabled or every redirect target revalidated
- [ ] Timeout set to 3 seconds or less
- [ ] Response size capped
- [ ] URL credentials rejected (`https://user:pass@host`)
- [ ] DNS rebinding considered
- [ ] Egress firewall blocks metadata and private IP ranges
- [ ] Blocked attempts logged with user ID, URL, resolved IP, and request ID

## PDPA 2024 Consequence

SSRF can expose cloud credentials, environment variables, internal admin panels, and personal data stores. Under PDPA 2024, failure to block obvious internal and metadata destinations is not a subtle vulnerability -- it is a failure to implement reasonable security measures under Seksyen 5(1) and 5(2), carrying up to RM1,000,000 fine or 3 years imprisonment.
