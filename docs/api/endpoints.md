# API / Endpoints Reference

openGPLMS does not have a formal REST API. All interactions happen through HTML forms (POST) or GET parameters. This document catalogs every form action and AJAX-style endpoint.

---

## AJAX Endpoints

### `POST forgot_message.php`

**Access:** Public

Accepts `application/x-www-form-urlencoded`. Returns JSON.

**Request Parameters:**

| Parameter | Required | Description |
|---|---|---|
| `username` | Yes | Username or full name of the person requesting a password reset |
| `email` | No | Contact email (optional, included in the message to the admin) |

**Response:**

```json
{ "ok": true }
```

Always returns `{"ok": true}` to prevent username enumeration, even if no admin is found or the request cannot be processed. Returns `{"ok": false}` only on unexpected server exceptions.

**Side effects:** Inserts a message in `{prefix}messages` to the first active admin, and logs the event in `{prefix}audit_log`.

---

## Form POST Actions

All form POST actions follow the Post-Redirect-Get pattern: every handler ends with a `Location` header redirect. This prevents duplicate submissions on browser refresh.

All POST forms include a CSRF token field. `verifyCsrf()` is called at the top of each POST handler.

---

### `index.php` — Login

| Field | Value |
|---|---|
| `action` | `login` |
| `username` | string |
| `password` | string |
| `csrf_token` | CSRF token |

On success: sets session, logs `login` action, redirects to `dashboard.php`.

On failure: increments rate limit counter, re-renders form with error message.

---

### `catalog_public.php` — Login (from public catalog)

Same fields as `index.php` login. The login modal on the public catalog submits to the same page via `action=login`. On success: redirects to `dashboard.php`.

---

### `catalog.php` — Book Actions

All actions POST to `catalog.php`.

#### `delete_book`
| Field | Value |
|---|---|
| `action` | `delete_book` |
| `delete_id` | Integer book ID |

Authorization: admin or book owner. Logs `delete` in audit log.

#### `mass_delete` (Admin only)
| Field | Value |
|---|---|
| `action` | `mass_delete` |
| `mass_ids[]` | Array of integer book IDs |

#### `mass_status` (Admin only)
| Field | Value |
|---|---|
| `action` | `mass_status` |
| `mass_ids[]` | Array of integer book IDs |
| `new_status` | One of: `Διαθέσιμο`, `Μη Διαθέσιμο`, `Σε Επεξεργασία` |

#### `mass_visibility` (Admin only)
| Field | Value |
|---|---|
| `action` | `mass_visibility` |
| `mass_ids[]` | Array of integer book IDs |
| `new_visibility` | `1` (public) or `0` (private) |

#### `request_permission`
| Field | Value |
|---|---|
| `action` | `request_permission` |
| `book_id` | Integer book ID |
| `req_action` | `edit` or `delete` |
| `reason` | String (required) |

Sends a message to all active admins. Logs `request_permission` in audit log.

---

### `add_book.php` — Create / Edit Book

| Field | Notes |
|---|---|
| `title` | Required |
| `author` | Required |
| `isbn` | Optional |
| `type` | ENUM value |
| `category_id` | Integer or empty |
| `publisher_id` | Integer or empty |
| `year` | Integer or empty |
| `language` | String |
| `pages` | Integer or empty |
| `edition` | String |
| `volume` | String |
| `location` | String |
| `description` | Text |
| `status` | One of the status ENUM values |
| `is_public` | Present = 1, absent = 0 (checkbox) |
| `cover_url` | URL string (sanitized server-side) |

In **edit mode** (URL has `?id=`): performs UPDATE and logs `edit`.
In **create mode**: performs INSERT and logs `create`. Redirects to `book.php?id=<new_id>`.

---

### `book.php` — Permission Request

| Field | Value |
|---|---|
| `action` | `request_permission` |
| `req_action` | `edit` or `delete` |
| `reason` | String (required) |

Sends message to the first active admin (not all admins, unlike `catalog.php`). Logs `request_permission`.

---

### `categories.php` — Category & Publisher Management

#### `add_cat`
| Field | Value |
|---|---|
| `action` | `add_cat` |
| `cat_name` | String (required) |
| `cat_desc` | String (optional) |

#### `del_cat`
| Field | Value |
|---|---|
| `action` | `del_cat` |
| `id` | Integer category ID |

#### `add_pub`
| Field | Value |
|---|---|
| `action` | `add_pub` |
| `pub_name` | String (required) |

#### `del_pub`
| Field | Value |
|---|---|
| `action` | `del_pub` |
| `id` | Integer publisher ID |

---

### `users.php` — User Management (Admin Only)

#### `create`
| Field | Value |
|---|---|
| `action` | `create` |
| `username` | String (required) |
| `password` | String ≥6 chars (required) |
| `role` | `admin`, `employee`, or `user` |

#### `toggle`
| Field | Value |
|---|---|
| `action` | `toggle` |
| `uid` | Integer user ID |

Flips `active` between 0 and 1 using `SET active=1-active`.

#### `change_role`
| Field | Value |
|---|---|
| `action` | `change_role` |
| `uid` | Integer user ID |
| `role` | `admin`, `employee`, or `user` |

#### `reset_pass`
| Field | Value |
|---|---|
| `action` | `reset_pass` |
| `uid` | Integer user ID |
| `new_password` | String ≥6 chars |

#### `delete`
| Field | Value |
|---|---|
| `action` | `delete` |
| `uid` | Integer user ID |

#### `send_message`
| Field | Value |
|---|---|
| `action` | `send_message` |
| `uid` | Integer user ID (recipient) |
| `msg_subject` | String (optional) |
| `msg_body` | String (required) |

---

### `messages.php` — Messaging

#### `send`
| Field | Value |
|---|---|
| `action` | `send` |
| `to_user` | Integer user ID |
| `subject` | String (optional) |
| `body` | String (required) |

#### `reply`
| Field | Value |
|---|---|
| `action` | `reply` |
| `to_user` | Integer user ID |
| `subject` | String (pre-filled as `Re: [original]`) |
| `body` | String (required) |

#### `delete`
| Field | Value |
|---|---|
| `action` | `delete` |
| `msg_id` | Integer message ID |

Authorization: only the sender or recipient of the message can delete it.

#### `mark_all_read`
| Field | Value |
|---|---|
| `action` | `mark_all_read` |

Sets `is_read=1` for all messages where `to_user = session user`.

---

### `profile.php` — Change Own Password

| Field | Value |
|---|---|
| `current_password` | Current password (verified with `password_verify`) |
| `new_password` | New password ≥6 chars |
| `confirm_password` | Must match `new_password` |

---

### `audit.php` — Backup / Export

#### CSV Export (POST)
| Field | Value |
|---|---|
| `action` | `export` |

Outputs the full audit log as a CSV file download.

#### Backup (POST)
| Field | Value |
|---|---|
| `action` | `backup` |

Currently non-functional — shows a warning modal before the form can be submitted.

---

### `reports.php` — Clear Audit Log / CSV Export

#### Clear Audit Log (POST, Admin Only)
| Field | Value |
|---|---|
| `action` | `clear_audit` |

Deletes all rows from `{prefix}audit_log`, then immediately inserts a `clear` entry.

#### Full Catalog CSV Export (GET)
```
GET reports.php?export=1
```
No POST fields. Outputs the full `{prefix}books` table joined with categories and publishers as a CSV download.

---

### `csv_import.php` — File Upload

| Field | Value |
|---|---|
| `csv_file` | File upload (`.csv` or `.txt`) |
| `delimiter` | `,` `,  `;`, or `\t` |
| `skip_header` | Present = skip first row |
| `update_existing` | Present = update records with matching ISBN |

Responds with a results page (not a redirect) showing per-row import status.

### Template Download (GET)
```
GET csv_import.php?template=1
```
Downloads a UTF-8 BOM CSV with headers and two example rows.
