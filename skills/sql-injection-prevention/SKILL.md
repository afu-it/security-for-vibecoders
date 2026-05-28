---
name: sql-injection-prevention
description: "Detect and fix SQL injection vulnerabilities in any framework. Covers Laravel (DB::raw, whereRaw), Node.js (template literals in queries), Python (f-strings in SQL), and Cloudflare D1. Enforces parameterized bindings everywhere. Use when writing database queries, reviewing code for injection, or fixing SQL injection findings."
version: "1.0"
---

# SQL Injection Prevention

Detect and fix SQL injection vulnerabilities across any framework. This is Finding #1 from Malaysian PDPA court cases -- the most common and most dangerous vulnerability.

## The Rule

**NEVER concatenate user input into SQL strings. ALWAYS use parameterized bindings.**

## Detection Patterns

### Node.js / TypeScript

BAD (vulnerable):
```typescript
// Template literal injection
const result = await db.query(`SELECT * FROM users WHERE id = '${userId}'`);

// String concatenation
const result = await db.query("SELECT * FROM users WHERE name = '" + name + "'");

// Tagged template without proper escaping
const result = await db.raw(`SELECT * FROM orders WHERE status = ${status}`);
```

GOOD (safe):
```typescript
// Parameterized binding with ?
const result = await db.prepare('SELECT * FROM users WHERE id = ?').bind(userId).first();

// Named parameters
const result = await db.query('SELECT * FROM users WHERE id = $1', [userId]);

// ORM with built-in escaping
const result = await prisma.user.findUnique({ where: { id: userId } });
```

### Cloudflare D1

BAD:
```typescript
await db.exec(`INSERT INTO logs (msg) VALUES ('${userInput}')`);
```

GOOD:
```typescript
await db.prepare('INSERT INTO logs (msg) VALUES (?)').bind(userInput).run();
await db.batch([
  db.prepare('UPDATE credits SET amount = amount - 1 WHERE user_id = ?').bind(userId),
  db.prepare('INSERT INTO transactions (user_id, amount) VALUES (?, ?)').bind(userId, -1),
]);
```

### Laravel / PHP

BAD:
```php
DB::raw("SELECT * FROM users WHERE email = '$email'");
DB::select("SELECT * FROM users WHERE id = " . $request->id);
$query->whereRaw("status = '$status'");
$query->orderByRaw($request->sort_column . ' ' . $request->sort_direction);
```

GOOD:
```php
DB::select('SELECT * FROM users WHERE email = ?', [$email]);
$query->whereRaw('status = ?', [$status]);
$query->orderByRaw('?? ??', [$sortColumn, $sortDirection]);
User::where('email', $email)->first();
```

### Python

BAD:
```python
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
cursor.execute("SELECT * FROM users WHERE name = '" + name + "'")
cursor.execute("SELECT * FROM users WHERE id = %s" % user_id)
```

GOOD:
```python
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
cursor.execute("SELECT * FROM users WHERE id = :id", {"id": user_id})
User.objects.filter(id=user_id).first()  # Django ORM
```

## Search Patterns

When auditing a codebase, search for these regex patterns:

```
# Node.js/TypeScript
`.*\$\{.*\}.*` near SELECT|INSERT|UPDATE|DELETE
".*" \+ .* near query|execute|raw
\.raw\(.*\$\{
\.exec\(`

# Laravel
DB::raw\(.*\$
whereRaw\(.*\$
orderByRaw\(.*\$
->select\(DB::raw\(.*\$

# Python
execute\(f"
execute\(".*" \+
execute\(".*%s" %
```

## Fix Process

1. **Find all database query locations** -- search for query/prepare/execute/raw calls
2. **Check each one** -- does it use `?` placeholders or string interpolation?
3. **Replace interpolation with bindings** -- swap `${var}` with `?` and add `.bind(var)`
4. **Verify with tests** -- run existing tests to ensure queries still work
5. **Add integration tests** -- test that malicious input is safely handled

## PDPA 2024 Consequence

**Seksyen 5(1) dan 5(2) PDPA 2010 (Pindaan 2024):**
- Denda sehingga RM1,000,000
- Penjara sehingga 3 tahun
- Atau kedua-duanya sekali

SQL injection is the #1 finding in forensic digital analysis of negligent systems.
