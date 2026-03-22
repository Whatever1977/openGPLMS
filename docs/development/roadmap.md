# Changelog & Roadmap

## Current Version: 0.1

Initial public release. All features described in this documentation are present in v0.1.

---

## Known Issues

### Backup Feature Non-Functional

The **Δημιουργία Αντιγράφου Ασφαλείας** button in `audit.php` shows a warning modal and does not perform a backup. The underlying code for ZIP creation and `mysqldump` is present in the file but the UI intentionally blocks it with the warning modal, indicating the feature is incomplete.

**Workaround:** Use phpMyAdmin or `mysqldump` directly from the command line to back up your database. The CSS export from `audit.php` can back up the audit log data.

### `pageWindow()` Duplicated Across Files

The `pageWindow()` helper function is defined identically in `audit.php`, `catalog.php`, `categories.php`, and `reports.php` rather than being extracted to `config.php`. This is a code duplication issue with no functional impact.

### `edit_book.php` Is a Redirect Proxy

`edit_book.php` does nothing except redirect to `add_book.php?id=X`. It exists for URL friendliness but adds a redirect hop. Bookmarks or links to `edit_book.php?id=X` work correctly.

### Backup Feature Uses `exec()`

The backup code in `audit.php` calls `exec()` to run `mysqldump`. Many shared hosting environments disable `exec()`. The code handles this gracefully (falls back to a README file inside the ZIP) but the backup ZIP will not contain the database dump in those environments.

### `removit/` Directory

The `src/removit/` directory contains `install.php` (drop-and-recreate), `login.php`, and `login.css`. The `login.php` and `login.css` files appear to be legacy files with no current function — only `install.php` is referenced in the documentation. These files should be reviewed and cleaned up.

### Login Rate Limiting is Session-Based

The login rate limiter stores state in `$_SESSION` rather than a database table. This means:
- Rate limiting is per-browser session, not strictly per-IP across sessions
- Clearing cookies or starting a new session resets the counter
- There is no persistent lockout across server restarts

This is acceptable for most small library deployments but should be noted for security-sensitive environments.

---

## Planned / Suggested Features

The following are potential enhancements based on the current codebase:

- **Email notifications** — currently the system only has internal messages; email integration for password resets would improve UX
- **Lending/circulation module** — track who has borrowed which item and due dates
- **Barcode scanner support** — integrate ISBN lookup from an external API (e.g. Open Library) to auto-fill book metadata
- **Export to PDF** — generate printable catalog reports or item labels
- **Automated backup** — a working scheduled backup via cron + mysqldump
- **Multi-library support** — separate collections with their own user pools
- **OPDS catalog feed** — an Atom-based API for e-reader apps
- **Dark mode** — toggle between light and dark themes

---

## Migration Notes

### From v0.1 to Future Versions

No migration tooling exists yet. If the database schema changes in a future version, a migration script will be provided alongside the release.

If upgrading manually, compare the `CREATE TABLE` statements in the new `install.php` against your existing schema and apply `ALTER TABLE` statements as needed.

Always back up your database before upgrading.
