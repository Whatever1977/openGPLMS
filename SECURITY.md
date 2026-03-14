# Security Policy

## Supported Versions

Security fixes are applied to the latest version of the `main` branch only.

| Version | Supported |
|---------|-----------|
| Latest (`main`) | ✅ |
| Older tags | ❌ |

---

## Reporting a Vulnerability

If you discover a security vulnerability in openGPLMS, please **do not open a public GitHub Issue**. Disclosing vulnerabilities publicly before a fix is available puts all deployments at risk.

Instead, report it privately by opening a [GitHub Security Advisory](https://github.com/PanagiotisKotsorgios/openGPLMS/security/advisories/new) on this repository.

Please include:

- A clear description of the vulnerability
- The affected file(s) and line numbers if known
- Steps to reproduce or a proof-of-concept (no need to exploit a live system)
- The potential impact

You can expect an acknowledgement within **7 days** and a resolution or status update within **30 days**, depending on the complexity of the issue.

---

## Security Design Notes

The following security controls are built into the application. Reports that bypass or weaken any of these are especially valuable:

- **CSRF protection** — every POST form includes a token verified server-side with `hash_equals()`.
- **Password hashing** — all passwords are stored as bcrypt hashes via `password_hash()` / `password_verify()`. Plaintext passwords are never stored or logged.
- **Login rate limiting** — 5 consecutive failures from the same IP trigger a 15-minute lockout.
- **Session hardening** — `httponly` and `samesite=Lax` flags are set; `strict_mode` rejects unknown session IDs.
- **Prepared statements** — all database queries use PDO prepared statements with bound parameters.
- **Role enforcement** — every page calls `requireAdmin()`, `requireEmployee()`, or `requireLogin()` before serving any content or processing any action.
- **URL sanitisation** — `cover_url` values are validated to allow only `http://` and `https://` schemes.
- **Installer exposure** — `install.php` and `removit/install.php` must be deleted after use. Leaving them accessible allows unauthenticated database recreation.

---

## Known Deployment Risks

- `install.php` and `removit/install.php` **must be deleted** after installation. These files are intentionally excluded from any production deployment checklist.
- `config.php` contains database credentials. Ensure your web server is configured to never serve `.php` source files directly and that `config.php` is not accessible from outside the web root.
- For HTTPS deployments, uncomment `ini_set('session.cookie_secure', 1)` in `config.php`.
