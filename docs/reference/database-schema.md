# Database Schema Reference

All tables use `InnoDB`, `utf8mb4` charset, and `utf8mb4_unicode_ci` collation. The table prefix is configurable (default `lib_`). This document uses `{prefix}` as a placeholder.

---

## Tables

### `{prefix}users`

Stores all user accounts.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | INT AUTO_INCREMENT | No | — | Primary key |
| `username` | VARCHAR(100) | No | — | Unique |
| `password` | VARCHAR(255) | No | — | bcrypt hash |
| `role` | ENUM('admin','employee','user') | No | `'user'` | |
| `active` | TINYINT(1) | No | `1` | 0 = deactivated |
| `created_at` | DATETIME | Yes | CURRENT_TIMESTAMP | |

Indexes: `PRIMARY (id)`, `UNIQUE (username)`

---

### `{prefix}books`

The main collection table. Stores all item types (books, magazines, manuscripts, etc.).

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | INT AUTO_INCREMENT | No | — | Primary key |
| `title` | VARCHAR(300) | No | — | |
| `author` | VARCHAR(200) | No | — | |
| `isbn` | VARCHAR(30) | Yes | NULL | ISBN-13 or ISSN |
| `type` | ENUM (see below) | No | `'Βιβλίο'` | |
| `category_id` | INT | Yes | NULL | FK → categories |
| `publisher_id` | INT | Yes | NULL | FK → publishers |
| `year` | INT | Yes | NULL | Publication year |
| `language` | VARCHAR(50) | Yes | `'Ελληνικά'` | |
| `pages` | INT | Yes | NULL | |
| `edition` | VARCHAR(50) | Yes | NULL | |
| `volume` | VARCHAR(50) | Yes | NULL | |
| `location` | VARCHAR(100) | Yes | NULL | Shelf/location code |
| `description` | TEXT | Yes | NULL | Notes or abstract |
| `cover_url` | VARCHAR(500) | Yes | NULL | Sanitized http/https URL only |
| `status` | ENUM('Διαθέσιμο','Μη Διαθέσιμο','Σε Επεξεργασία') | No | `'Διαθέσιμο'` | |
| `is_public` | TINYINT(1) | Yes | `1` | 1 = visible in public catalog |
| `created_by` | INT | Yes | NULL | FK → users (no constraint) |
| `created_at` | DATETIME | Yes | CURRENT_TIMESTAMP | |
| `updated_at` | DATETIME | Yes | CURRENT_TIMESTAMP ON UPDATE | Auto-updates on every write |

**Type ENUM values:** `Βιβλίο`, `Περιοδικό`, `Εφημερίδα`, `Χειρόγραφο`, `Ημερολόγιο`, `Επιστολή`, `Άλλο`

Indexes: `PRIMARY (id)`, `KEY (category_id)`, `KEY (publisher_id)`, `KEY idx_books_created_by (created_by)`

Foreign keys:
- `category_id` → `{prefix}categories(id)` **ON DELETE SET NULL** (deleting a category orphans its books, not deletes them)
- `publisher_id` → `{prefix}publishers(id)` **ON DELETE SET NULL** (same behavior)

Note: `created_by` has no foreign key constraint — it is intentionally left without one to allow audit trails even after the creating user is deleted.

---

### `{prefix}categories`

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | INT AUTO_INCREMENT | No | — | Primary key |
| `name` | VARCHAR(100) | No | — | Unique |
| `description` | TEXT | Yes | NULL | |
| `created_at` | DATETIME | Yes | CURRENT_TIMESTAMP | |

Indexes: `PRIMARY (id)`, `UNIQUE (name)`

---

### `{prefix}publishers`

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | INT AUTO_INCREMENT | No | — | Primary key |
| `name` | VARCHAR(200) | No | — | Unique |
| `created_at` | DATETIME | Yes | CURRENT_TIMESTAMP | |

Indexes: `PRIMARY (id)`, `UNIQUE (name)`

---

### `{prefix}messages`

Internal messaging between users.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | INT AUTO_INCREMENT | No | — | Primary key |
| `from_user` | INT | Yes | NULL | FK → users; NULL if sender deleted |
| `to_user` | INT | Yes | NULL | FK → users; message deleted if recipient deleted |
| `subject` | VARCHAR(200) | Yes | NULL | |
| `body` | TEXT | Yes | NULL | |
| `is_read` | TINYINT(1) | Yes | `0` | 0 = unread |
| `created_at` | DATETIME | Yes | CURRENT_TIMESTAMP | |

Indexes: `PRIMARY (id)`, `KEY (from_user)`, `KEY (to_user)`

Foreign keys:
- `from_user` → `{prefix}users(id)` **ON DELETE SET NULL** — message stays but sender becomes NULL
- `to_user` → `{prefix}users(id)` **ON DELETE CASCADE** — deleting a user deletes their received messages

---

### `{prefix}audit_log`

Immutable action log. Rows are never updated, only inserted and (bulk-)deleted.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | INT AUTO_INCREMENT | No | — | Primary key |
| `user_id` | INT | Yes | NULL | Who performed the action; NULL for system events |
| `action` | VARCHAR(100) | Yes | NULL | Action slug (see below) |
| `target_type` | VARCHAR(50) | Yes | NULL | Entity type affected |
| `target_id` | INT | Yes | NULL | ID of the affected entity |
| `details` | TEXT | Yes | NULL | Human-readable extra info |
| `created_at` | DATETIME | Yes | CURRENT_TIMESTAMP | |

Indexes: `PRIMARY (id)`

No foreign keys on `user_id` — this is intentional. The audit log must preserve historical records even if the user who performed an action is later deleted.

**Known action slugs:**

| Slug | Meaning |
|---|---|
| `login` | User logged in |
| `logout` | User logged out |
| `create` | Book created |
| `edit` | Book edited |
| `delete` | Book deleted |
| `mass_delete` | Admin bulk-deleted books |
| `mass_status` | Admin bulk-changed status |
| `mass_visibility` | Admin bulk-changed is_public |
| `csv_import` | CSV import completed |
| `create_user` | New user created |
| `delete_user` | User deleted |
| `change_role` | User role changed |
| `reset_password` | User password reset by admin |
| `change_password` | User changed own password |
| `send_message` | Internal message sent |
| `request_permission` | Employee requested edit/delete permission |
| `forgot_password_request` | Forgot-password request submitted |
| `backup` | System backup created |
| `clear` | Audit log cleared |

---

## Entity Relationship Summary

```
users ──────────────────────────────────────────────────┐
  │ id                                                   │
  │                                                      │
  ├─── created_by ──────────────── books ───── category_id ──── categories
  │                                  │
  │                                  └───── publisher_id ──── publishers
  │
  ├─── from_user / to_user ─── messages
  │
  └─── user_id ─────────────── audit_log
```

- Users → Books: one-to-many (via `created_by`, no FK constraint)
- Categories → Books: one-to-many (FK with SET NULL)
- Publishers → Books: one-to-many (FK with SET NULL)
- Users → Messages: many-to-many (via `from_user` / `to_user`)
- Users → Audit Log: one-to-many (via `user_id`, no FK constraint)

---

## Seed Data (from `install.php`)

The installer inserts:

**Users:**
- `admin` (role: admin, active: 1)
- `employee` (role: employee, active: 1)

**Categories:** Ιστορία, Λογοτεχνία, Επιστήμη

**Publishers:** Εκδόσεις Καστανιώτη, Εκδόσεις Πατάκη, Εκδόσεις Μεταίχμιο

**Books (5 items):**
1. Ιστορία του Ελληνικού Έθνους (Βιβλίο)
2. Ζορμπάς ο Έλληνας (Βιβλίο)
3. Εισαγωγή στη Φυσική (Βιβλίο)
4. Αρχαιολογία & Ιστορία (Περιοδικό)
5. Χειρόγραφες Σημειώσεις Βυζαντινής Μουσικής (Χειρόγραφο, status: Μη Διαθέσιμο)
