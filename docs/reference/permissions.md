# Role & Permission Model

openGPLMS has three roles. Access is enforced at two levels: page-level guards and record-level authorization.

---

## Role Matrix

| Feature / Page | Admin | Employee | User (no login) |
|---|:---:|:---:|:---:|
| Public catalog (`catalog_public.php`) | ✓ | ✓ | ✓ |
| Dashboard | ✓ | ✓ | ✗ |
| Staff catalog (view) | ✓ | ✓ | ✗ |
| Add book | ✓ | ✓ | ✗ |
| Edit own book | ✓ | ✓ | ✗ |
| Edit any book | ✓ | Request only | ✗ |
| Delete own book | ✓ | Request only* | ✗ |
| Delete any book | ✓ | Request only | ✗ |
| Mass delete / status / visibility | ✓ | ✗ | ✗ |
| CSV import | ✓ | ✓ | ✗ |
| CSV editor | ✓ | ✓ | ✗ |
| Reports & statistics | ✓ | ✓ | ✗ |
| Clear audit log | ✓ | ✗ | ✗ |
| Full audit log (`audit.php`) | ✓ | ✗ | ✗ |
| Categories & Publishers | ✓ | ✗ | ✗ |
| User management | ✓ | ✗ | ✗ |
| Internal messaging | ✓ | ✓ | ✗ |
| Profile (own password) | ✓ | ✓ | ✗ |
| Help / FAQ | ✓ | ✓ | ✗ |

\* Employees who created a book can click the **Διαγραφή (Αίτημα)** button, which sends a message to the admin rather than performing a direct delete.

---

## Page-Level Guards

Each page calls one of three guards as its first line of business logic:

```php
requireAdmin();     // admin only
requireEmployee();  // admin or employee
requireLogin();     // any logged-in user (used by none currently — all pages use requireEmployee minimum)
```

These functions check the session role and redirect immediately if the check fails — no page content is rendered for unauthorized users.

---

## Record-Level Authorization (Books)

For books, ownership is determined by the `created_by` column:

```php
$isOwner = ((int)($book['created_by'] ?? 0) === (int)($_SESSION['user_id'] ?? 0));
$canManage = isAdmin() || $isOwner;
```

This check is performed in both `book.php` (detail view) and `catalog.php` (catalog row). The UI adapts based on `$canManage`:

- If `$canManage` is `true`: show direct edit/delete buttons
- If `$canManage` is `false`: show "edit request" and "delete request" buttons instead

The server-side enforcement in `catalog.php` POST handler re-verifies ownership before executing any deletion, regardless of what the client sends.

---

## Permission Request Flow

When an employee does not have `canManage` access to a book:

1. Employee clicks **Επεξεργασία (Αίτημα)** or **Διαγραφή (Αίτημα)**
2. A modal prompts for a reason (required)
3. On submit, a POST to `catalog.php` (or `book.php`) with `action=request_permission`
4. The server inserts a message into `{prefix}messages` addressed to all active admins (catalog.php) or the first active admin (book.php)
5. The message body contains the employee's name, the requested action, the book's title/ID/ISBN, and the provided reason
6. The action is logged in the audit log with `action='request_permission'`
7. The admin sees the request in their message inbox and can act on it manually

---

## Admin Self-Protection Rules

Admins are protected from accidentally locking themselves out:

| Action | Blocked For |
|---|---|
| Deactivate user | Current admin's own account |
| Change role | Current admin's own account |
| Delete user | Current admin's own account |

These checks are enforced server-side in `users.php`:
```php
if ($uid === (int)$_SESSION['user_id']) {
    flash('Δεν μπορείτε να [action] τον εαυτό σας.', 'error');
    header('Location: users.php'); exit;
}
```

---

## Session Variables

After login, the following session keys are set:

| Key | Value |
|---|---|
| `$_SESSION['user_id']` | Integer ID from `{prefix}users.id` |
| `$_SESSION['username']` | Username string |
| `$_SESSION['role']` | One of: `'admin'`, `'employee'`, `'user'` |
| `$_SESSION['csrf_token']` | 64-character hex CSRF token |
| `$_SESSION['flash']` | Array `['msg', 'type']` — consumed on next page load |
| `$_SESSION['login_attempts_*']` | Rate limiting state per IP (keyed by `md5($ip)`) |
