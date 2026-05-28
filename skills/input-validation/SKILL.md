---
name: input-validation
description: "Validate and sanitize all user input to prevent XSS, path traversal, command injection, and malformed data. Covers Zod, Joi, class-validator, Laravel validation, and Python Pydantic. Use when building forms, API endpoints, file uploads, or fixing injection vulnerabilities beyond SQL."
version: "1.0"
---

# Input Validation

Validate and sanitize ALL user input at every entry point. This covers Finding #2 (path traversal) and extends beyond SQL injection to XSS, command injection, and data integrity.

## The Rule

**NEVER trust user input. ALWAYS validate type, length, format, and range on the server side.**

## Attack Vectors Prevented

| Attack | How it works | Validation fix |
|--------|-------------|----------------|
| XSS | Inject `<script>` into rendered HTML | HTML-encode output, validate no HTML in text fields |
| Path Traversal | `../../etc/passwd` in file paths | Whitelist allowed paths, reject `..` |
| Command Injection | `; rm -rf /` in shell commands | Never pass user input to shell, use spawn with args array |
| SSRF | User-controlled URLs fetch internal services | Whitelist allowed domains, block private IPs |
| Prototype Pollution | `__proto__` in JSON keys | Strip dangerous keys, use `Object.create(null)` |
| File Upload Abuse | `.php` file uploaded as "image" | Validate MIME type + magic bytes, not just extension |

## Framework Patterns

### Node.js with Zod (Recommended)

```typescript
import { z } from 'zod';

// Define strict schemas
const CreateUserSchema = z.object({
  email: z.string().email().max(255),
  name: z.string().min(1).max(100).regex(/^[a-zA-Z\s'-]+$/),
  age: z.number().int().min(13).max(150),
  role: z.enum(['user', 'admin']),
});

// Validate at API boundary
app.post('/users', async (req, res) => {
  const result = CreateUserSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(400).json({
      error: 'Validation failed',
      issues: result.error.issues.map(i => ({
        field: i.path.join('.'),
        message: i.message,
      })),
    });
  }
  const validated = result.data; // Type-safe, validated
  await createUser(validated);
});
```

### Path Traversal Prevention

BAD:
```typescript
app.get('/files/:filename', (req, res) => {
  const file = path.join('/uploads', req.params.filename);
  res.sendFile(file); // ../../etc/passwd works!
});
```

GOOD:
```typescript
import path from 'path';

const UPLOAD_DIR = path.resolve('/uploads');

app.get('/files/:filename', (req, res) => {
  const filename = path.basename(req.params.filename); // Strip directory traversal
  const filePath = path.resolve(UPLOAD_DIR, filename);

  // Verify resolved path is still within allowed directory
  if (!filePath.startsWith(UPLOAD_DIR)) {
    return res.status(403).json({ error: 'Access denied' });
  }

  res.sendFile(filePath);
});
```

### XSS Prevention

BAD:
```typescript
// Server-side rendering with raw user input
app.get('/profile', (req, res) => {
  res.send(`<h1>Welcome, ${req.query.name}</h1>`);
});
```

GOOD:
```typescript
import { encode } from 'html-entities';

app.get('/profile', (req, res) => {
  const safeName = encode(req.query.name || '');
  res.send(`<h1>Welcome, ${safeName}</h1>`);
});

// Or better: use a template engine with auto-escaping (EJS, Handlebars, React)
```

### File Upload Validation

```typescript
import { fileTypeFromBuffer } from 'file-type';

const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp'];
const MAX_SIZE = 5 * 1024 * 1024; // 5MB

async function validateUpload(file: Buffer, filename: string) {
  // Check size
  if (file.length > MAX_SIZE) {
    throw new Error('File too large (max 5MB)');
  }

  // Check actual file type via magic bytes (not extension!)
  const type = await fileTypeFromBuffer(file);
  if (!type || !ALLOWED_TYPES.includes(type.mime)) {
    throw new Error(`Invalid file type. Allowed: ${ALLOWED_TYPES.join(', ')}`);
  }

  // Generate safe filename (never use user-provided filename directly)
  const safeFilename = `${crypto.randomUUID()}.${type.ext}`;
  return safeFilename;
}
```

### Command Injection Prevention

BAD:
```typescript
import { exec } from 'child_process';
exec(`convert ${userFilename} output.png`); // Shell injection!
```

GOOD:
```typescript
import { execFile } from 'child_process';
// execFile does NOT use shell, so injection is impossible
execFile('convert', [userFilename, 'output.png']);
```

### Laravel / PHP

```php
// Form request validation
class StoreUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'email' => ['required', 'email:rfc,dns', 'max:255'],
            'name' => ['required', 'string', 'max:100', 'regex:/^[a-zA-Z\s\'-]+$/'],
            'age' => ['required', 'integer', 'min:13', 'max:150'],
            'role' => ['required', Rule::in(['user', 'admin'])],
            'avatar' => ['nullable', 'image', 'mimes:jpg,png,webp', 'max:5120'],
        ];
    }
}
```

### Python with Pydantic

```python
from pydantic import BaseModel, EmailStr, Field, validator
import re

class CreateUser(BaseModel):
    email: EmailStr
    name: str = Field(min_length=1, max_length=100)
    age: int = Field(ge=13, le=150)
    role: Literal['user', 'admin']

    @validator('name')
    def name_must_be_safe(cls, v):
        if not re.match(r"^[a-zA-Z\s'-]+$", v):
            raise ValueError('Name contains invalid characters')
        return v

# Usage in FastAPI
@app.post("/users")
async def create_user(user: CreateUser):
    # user is already validated and type-safe
    pass
```

## Validation Checklist

For every API endpoint, verify:

- [ ] All input fields have type validation (string, number, boolean)
- [ ] String fields have max length limits
- [ ] Numeric fields have min/max range
- [ ] Enum fields use allowlists (not blocklists)
- [ ] Email fields use proper email validation
- [ ] URL fields validate protocol (https only) and domain
- [ ] File uploads check magic bytes, not just extension
- [ ] Path parameters cannot traverse directories
- [ ] No user input is passed to shell commands
- [ ] No user input is used in dynamic `import()` or `eval()`

## PDPA 2024 Consequence

**Seksyen 5(1) dan 5(2) PDPA 2010 (Pindaan 2024):**
- Denda sehingga RM1,000,000
- Penjara sehingga 3 tahun

Path traversal (Finding #2) allowed attackers to download the entire user database. The developer was charged because input validation is a **basic security measure** that any competent developer should implement.
