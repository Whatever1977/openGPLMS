# Security Guide

openGPLMS was built with security as a primary concern. This document describes every security mechanism in the codebase and gives advice for production hardening.

---

## Authentication

### Password Hashing

All passwords are stored using PHP's `password_hash()` with `PASSWORD_DEFAULT` (bcrypt). Plaintext passwords are never stored or logged. Password verification uses `password_verify()`.

Minimum password length is 6 characters (enforced both client-side with `minlength` and server-side in `users.php` and `profile.php`).

### Session Security

Session settings are configured in `config.php` before `session_start()`:

```php
ini_set('session.cookie_httponly', 1);    // Blocks JavaScript access to the cookie
ini_set('session.use_strict_mode', 1);    // Rejects unknown session IDs (prevents fixation)
ini_set('session.cookie_samesite', 'Lax'); // Mitigates cross-site request timing
// ini_set('session.cookie_secure', 1);   // Uncomment on HTTPS
```

On successful login, `session_regenerate_id(true)` is called to issue a new session ID and invalidate the old one, preventing session fixation attacks.

On logout, `session_destroy()` is called.

### Login Rate Limiting

Failed login attempts are tracked per IP address in the session (no extra table needed):

- **5 failed attempts** → 15-minute lockout for that IP
- The remaining attempt count is shown in the error message
- On successful login, the counter is cleared via `loginClearAttempts()`

The relevant functions in `config.php`: `loginCheckRateLimit()`, `loginRecordFailure()`, `loginClearAttempts()`, `loginRemainingSeconds()`.

---

## CSRF Protection

Every state-changing POST form includes a CSRF token. The token is generated once per session via `csrfToken()` (using `random_bytes(32)` → `bin2hex`) and stored in `$_SESSION['csrf_token']`.

Usage in forms:
```php
echo csrfField();
// Outputs: <input type="hidden" name="csrf_token" value="...">
```

Verification at the top of every POST handler:
```php
verifyCsrf();
// Calls hash_equals() for timing-safe comparison
// Terminates with HTTP 403 on mismatch
```

`hash_equals()` is used (not `==`) to prevent timing-based token extraction attacks.

---

## SQL Injection Prevention

All user input reaches the database through PDO prepared statements with positional `?` parameters. There is **zero raw string interpolation** of user-controlled values into SQL queries.

Dynamic `ORDER BY` in `catalog_public.php` uses a strict whitelist map:
```php
$sortOptions = [
    'title_asc'  => 'b.title ASC',
    'title_desc' => 'b.title DESC',
    ...
];
$orderBy = $sortOptions[$sort] ?? 'b.title ASC';
```

Dynamic `LIMIT` and `OFFSET` values are always explicitly cast to `(int)` before interpolation.

---

## XSS Prevention

All output to HTML is escaped via the helper function:
```php
function h($s) {
    return htmlspecialchars($s, ENT_QUOTES, 'UTF-8');
}
```

The only exceptions are intentional HTML in static strings (e.g. the FAQ answers in `help.php`) which are written by developers, not users.

`data-*` attributes in the public catalog also pass through `h()`.

---

## Cover URL Sanitization

The `cover_url` field accepts external image URLs. To prevent `javascript:` or `data:` URI injection via image attributes:

```php
function sanitizeCoverUrl(string $url): string {
    $url = trim($url);
    if ($url === '') return '';
    if (!preg_match('#^https?://#i', $url)) return '';
    if (!filter_var($url, FILTER_VALIDATE_URL)) return '';
    return $url;
}
```

Only `http://` and `https://` absolute URLs are accepted.

---

## Role-Based Authorization

Every page enforces its minimum required role at the very top, before any business logic runs:

| Function | Minimum Role | Redirect on Failure |
|---|---|---|
| `requireLogin()` | Any logged-in user | `index.php` |
| `requireEmployee()` | `employee` or `admin` | `index.php` |
| `requireAdmin()` | `admin` only | `dashboard.php` |

Record-level authorization (who can edit/delete a specific book) is checked in `book.php` and `catalog.php`:

```php
$canManage = isAdmin() || ($ownerId > 0 && $ownerId === $meId);
```

Employees can only modify records where `created_by == $_SESSION['user_id']`. For all other records they must submit a permission request.

---

## Input Validation

- Required fields (title, author) are validated both client-side (`required` attribute) and server-side
- Enum-type fields (item type, status, visibility) use whitelists rather than accepting arbitrary strings
- Integer IDs from URL parameters are cast with `(int)` before use
- LIMIT/OFFSET values from GET parameters are validated against an allowed list: `[10, 25, 50]`
- Role strings are validated against `['admin', 'employee', 'user']` before any DB write

---

## Sensitive File Exposure

### `install.php` / `removit/install.php`

These files drop and recreate the entire database. They **must be deleted** after installation. There is no authentication check protecting them.

### `config.php`

Contains database credentials. It should not be web-accessible if placed outside the web root, but in a standard XAMPP/Apache setup it is readable via PHP (not as raw text). Do not expose it to public directory listing.

### `src/backups/`

If the backup feature is used, ZIP files are written to `src/backups/`. This directory should be protected from public web access via `.htaccess` or by being outside the web root.

---

## Production Checklist

- [ ] Delete `install.php` and `removit/install.php`
- [ ] Change default admin and employee passwords
- [ ] Uncomment `session.cookie_secure` in `config.php` (HTTPS only)
- [ ] Protect `src/backups/` from public web access
- [ ] Set `display_errors = Off` in `php.ini`
- [ ] Consider moving `config.php` above the web root and referencing it with an absolute path
- [ ] Keep PHP and MySQL up to date

---

## Audit Log

Every significant action is written to `{prefix}audit_log`:

- Login / logout
- Create / edit / delete book
- CSV import
- User create / delete / role change / password reset
- Permission requests
- Message sent

The log stores: `user_id`, `action`, `target_type`, `target_id`, `details`, `created_at`. The `user_id` column allows `NULL` to accommodate system-level entries.

The audit log can be exported as CSV from `audit.php` or cleared (with a confirmation step) from `reports.php` (admin only).
