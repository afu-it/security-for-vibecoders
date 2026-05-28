---
name: data-encryption
description: "Encrypt personal data at rest and in transit to comply with PDPA 2024. Covers field-level encryption for PII, AES-256-GCM patterns, key management, and database column encryption for Node.js, Python, Laravel, and Cloudflare Workers. Use when storing sensitive user data, implementing encryption, or responding to PDPA compliance requirements."
version: "1.0"
---

# Data Encryption

Encrypt personal data at rest to comply with PDPA 2024. If your database is breached but PII fields are encrypted, you have a strong defense against negligence charges.

## The Rule

**ENCRYPT all Personally Identifiable Information (PII) at rest. Use AES-256-GCM with proper key management.**

## What Must Be Encrypted

Under PDPA 2024, these fields are **data peribadi** (personal data) and should be encrypted at rest:

| Field | Why |
|-------|-----|
| IC number (NRIC) | Unique identifier, identity theft risk |
| Phone number | Direct contact, harassment risk |
| Home address | Physical safety risk |
| Bank account / card numbers | Financial fraud risk |
| Medical records | Discrimination risk |
| Biometric data | Cannot be changed if leaked |
| Email (in some contexts) | Phishing, account takeover |

## Implementation Patterns

### Node.js / TypeScript

```typescript
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

const ENCRYPTION_KEY = Buffer.from(env.ENCRYPTION_KEY, 'hex'); // 32 bytes for AES-256
const ALGORITHM = 'aes-256-gcm';

interface EncryptedField {
  ciphertext: string; // hex
  iv: string;         // hex
  tag: string;        // hex
}

function encrypt(plaintext: string): EncryptedField {
  const iv = randomBytes(12); // 96-bit IV for GCM
  const cipher = createCipheriv(ALGORITHM, ENCRYPTION_KEY, iv);

  let ciphertext = cipher.update(plaintext, 'utf8', 'hex');
  ciphertext += cipher.final('hex');
  const tag = cipher.getAuthTag();

  return {
    ciphertext,
    iv: iv.toString('hex'),
    tag: tag.toString('hex'),
  };
}

function decrypt(encrypted: EncryptedField): string {
  const decipher = createDecipheriv(
    ALGORITHM,
    ENCRYPTION_KEY,
    Buffer.from(encrypted.iv, 'hex')
  );
  decipher.setAuthTag(Buffer.from(encrypted.tag, 'hex'));

  let plaintext = decipher.update(encrypted.ciphertext, 'hex', 'utf8');
  plaintext += decipher.final('utf8');
  return plaintext;
}

// Usage with database
async function createUser(data: { name: string; ic_number: string; phone: string }) {
  const encryptedIC = encrypt(data.ic_number);
  const encryptedPhone = encrypt(data.phone);

  await db.prepare(`
    INSERT INTO users (name, ic_number_enc, ic_number_iv, ic_number_tag, phone_enc, phone_iv, phone_tag)
    VALUES (?, ?, ?, ?, ?, ?, ?)
  `).bind(
    data.name,
    encryptedIC.ciphertext, encryptedIC.iv, encryptedIC.tag,
    encryptedPhone.ciphertext, encryptedPhone.iv, encryptedPhone.tag,
  ).run();
}
```

### Cloudflare Workers (Web Crypto API)

```typescript
const ALGORITHM = { name: 'AES-GCM', length: 256 };

async function getKey(env: Env): Promise<CryptoKey> {
  const keyData = Buffer.from(env.ENCRYPTION_KEY, 'hex');
  return crypto.subtle.importKey('raw', keyData, ALGORITHM, false, ['encrypt', 'decrypt']);
}

async function encrypt(plaintext: string, env: Env): Promise<string> {
  const key = await getKey(env);
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const encoded = new TextEncoder().encode(plaintext);

  const ciphertext = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv },
    key,
    encoded
  );

  // Pack IV + ciphertext (GCM tag is appended automatically)
  const packed = new Uint8Array(iv.length + new Uint8Array(ciphertext).length);
  packed.set(iv);
  packed.set(new Uint8Array(ciphertext), iv.length);

  return Buffer.from(packed).toString('base64');
}

async function decrypt(packed: string, env: Env): Promise<string> {
  const key = await getKey(env);
  const data = Buffer.from(packed, 'base64');

  const iv = data.slice(0, 12);
  const ciphertext = data.slice(12);

  const plaintext = await crypto.subtle.decrypt(
    { name: 'AES-GCM', iv },
    key,
    ciphertext
  );

  return new TextDecoder().decode(plaintext);
}
```

### Python

```python
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os, base64

# Option 1: Fernet (simpler, includes timestamp)
key = os.environ['ENCRYPTION_KEY'].encode()  # base64-encoded 32 bytes
f = Fernet(key)

encrypted = f.encrypt(plaintext.encode())
decrypted = f.decrypt(encrypted).decode()

# Option 2: AES-256-GCM (more control)
key = bytes.fromhex(os.environ['ENCRYPTION_KEY'])  # 32 bytes hex
aesgcm = AESGCM(key)

def encrypt_field(plaintext: str) -> str:
    nonce = os.urandom(12)
    ciphertext = aesgcm.encrypt(nonce, plaintext.encode(), None)
    return base64.b64encode(nonce + ciphertext).decode()

def decrypt_field(packed: str) -> str:
    data = base64.b64decode(packed)
    nonce, ciphertext = data[:12], data[12:]
    return aesgcm.decrypt(nonce, ciphertext, None).decode()
```

### Laravel

```php
// Laravel has built-in encryption via APP_KEY
use Illuminate\Support\Facades\Crypt;

// In your model -- auto-encrypt/decrypt
class User extends Model
{
    protected $casts = [
        'ic_number' => 'encrypted',
        'phone' => 'encrypted',
        'address' => 'encrypted',
    ];
}

// Manual usage
$encrypted = Crypt::encryptString($icNumber);
$decrypted = Crypt::decryptString($encrypted);
```

## Key Management Rules

1. **Generate keys properly:**
   ```bash
   # Generate a 256-bit key
   openssl rand -hex 32
   ```

2. **Store keys separately from data** -- if DB is breached, keys should not be in the same place

3. **Rotate keys periodically** -- support multiple active keys:
   ```typescript
   const KEYS = {
     current: env.ENCRYPTION_KEY_V2,
     previous: env.ENCRYPTION_KEY_V1, // For decrypting old data
   };
   ```

4. **Never hardcode keys** -- always use environment variables or secret managers

5. **Use envelope encryption for large datasets** -- encrypt data with a data key, encrypt the data key with a master key

## Database Schema Pattern

```sql
-- Store encrypted fields with their IV/nonce
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT NOT NULL,

  -- Encrypted PII fields
  ic_number_enc TEXT,      -- Encrypted ciphertext
  phone_enc TEXT,          -- Encrypted ciphertext
  address_enc TEXT,        -- Encrypted ciphertext

  -- Key version for rotation support
  encryption_key_version INTEGER DEFAULT 1,

  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- You can still search by non-encrypted fields
-- For searchable encrypted fields, store a blind index (HMAC hash)
```

## Encryption Checklist

- [ ] All PII fields encrypted at rest (IC, phone, address, bank details)
- [ ] Using AES-256-GCM (authenticated encryption)
- [ ] Unique IV/nonce per encryption operation (never reuse)
- [ ] Encryption keys stored in environment variables, not code
- [ ] Key rotation strategy in place
- [ ] Keys stored separately from encrypted data
- [ ] HTTPS enforced for all data in transit
- [ ] Database backups are also encrypted

## PDPA 2024 Consequence

**Seksyen 5(1) dan 5(2) PDPA 2010 (Pindaan 2024):**
- Denda sehingga RM1,000,000
- Penjara sehingga 3 tahun

PDPA 2024 explicitly requires **langkah keselamatan yang munasabah** for protecting personal data. Encryption at rest is considered a baseline reasonable measure. If a breach occurs and data was stored in plaintext, the developer cannot argue they took reasonable precautions.
