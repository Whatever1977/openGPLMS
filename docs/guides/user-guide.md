# User Guide

This guide covers day-to-day use of openGPLMS for each role.

---

## Roles Overview

| Role | Can Do |
|---|---|
| **Admin** | Everything: full catalog CRUD, user management, categories, publishers, audit log, mass actions |
| **Employee** | Add and edit their own catalog entries, CSV import, reports, internal messaging, permission requests |
| **User** | View the public catalog only (no login required for this) |

---

## Logging In

Visit `index.php`. Click **Διαχείριση** (Management) or **Staff Login** in the footer to open the login modal. Enter your username and password.

After 5 failed attempts the system locks out your IP for 15 minutes. Contact an admin if locked out.

If you have forgotten your password, use the **"Ξεχάσατε τον κωδικό;"** link in the modal. This sends a message to the admin's inbox; they will reset it manually.

---

## Dashboard (`dashboard.php`)

The dashboard shows:
- Summary cards: total items, new this month, distinct item types, total users
- A doughnut chart of items by type
- A bar chart of monthly additions (last 6 months)
- A table of the 5 most recently added items

---

## Catalog Management

### Viewing the Catalog (`catalog.php`)

The staff catalog shows all items. Filter by:
- Free-text search (title, author, ISBN)
- Type (Book, Magazine, etc.)
- Category
- Language
- Year range
- Items per page (10 / 25 / 50)

Results are paginated with a sliding page-window control.

Export the current filtered results to CSV using the **Εξαγωγή CSV** button.

### Adding an Item (`add_book.php`)

Click **Προσθήκη Αντικειμένου**. Required fields are **Title** and **Author/Creator**. All other fields are optional.

Key fields:

| Field | Notes |
|---|---|
| Type | Chosen from the ENUM values in the database (Book, Magazine, Newspaper, Manuscript, Diary, Letter, Other) |
| Category / Publisher | Searchable dropdowns; selected from existing records |
| Status | Available / Unavailable / In Processing |
| Public | Toggle to show/hide from the public catalog |
| Cover URL | Must be an `http://` or `https://` URL; `javascript:` and `data:` schemes are rejected |

After saving, you are redirected to the single-item view.

### Editing an Item

From the catalog or item detail page, click the **pencil icon** or **Επεξεργασία** button.

**Who can edit?**
- Admins can edit any item.
- Employees can only edit items they created (`created_by == session user_id`).
- Employees who did not create an item can click **Επεξεργασία (Αίτημα)** to send a permission request to all admins via the internal message system.

### Deleting an Item

- Admins can delete any item directly (a confirmation modal appears).
- Employees who own the item can request deletion via **Διαγραφή (Αίτημα)**.
- The deletion action is permanent and is logged in the audit log.

### Admin Mass Actions

When logged in as admin, a checkbox appears on every row in the catalog table. Selecting one or more rows reveals the **mass actions bar**:

- **Αλλαγή Κατάστασης** — change status for all selected items at once
- **Ορισμός Δημοσιότητας** — set all selected items to Public or Private
- **Μαζική Διαγραφή** — permanently delete all selected items (a detailed confirmation modal shows the titles before proceeding)

---

## Categories & Publishers (`categories.php`)

Admin-only. Two side-by-side panels with searchable, paginated tables.

- Add a new category (name + optional description) via the **Νέα** button
- Add a new publisher via **Νέος**
- Delete a category or publisher using the trash icon (a browser `confirm()` dialog appears)

Deleting a category or publisher does **not** delete associated books — it sets `category_id` or `publisher_id` to `NULL` on those records.

---

## User Management (`users.php`)

Admin-only.

### Creating a User

Click **Νέος Χρήστης**. Fill in username, password (≥6 characters), and role. Use the shuffle button to generate a random 12-character password.

### Managing Users

Each row in the user table has:
- **Role select** — change role instantly with `onchange` submit (admins cannot change their own role)
- **Toggle switch** — activate/deactivate (admins cannot deactivate themselves)
- **Key icon** — reset the user's password
- **Chat icon** — send a direct message to the user
- **Trash icon** — permanently delete the user (not available for the current admin)

User stats (book count, message count, last activity) are shown inline.

---

## Internal Messaging (`messages.php`)

The messaging system supports employee-to-employee and employee-to-admin communication.

**Inbox** shows messages sent to you. **Απεσταλμένα** shows messages you sent.

Click any message to open the read pane. You can reply (only if you are the recipient) or delete the message.

Use **Νέο Μήνυμα** to compose. Recipients are grouped by role in the dropdown.

A badge on the sidebar link shows unread message count.

---

## CSV Import

See the dedicated [CSV Import & Editor Guide](csv-import.md).

---

## Reports & Statistics (`reports.php`)

Available to all employees. Shows:
- Summary cards (total items, new this month, type variety)
- Four Chart.js charts: item type distribution, monthly additions history, items by category, items by language
- A paginated audit log table with search and per-page controls

Admins can **clear the audit log** from this page. The clear action is itself logged immediately after.

Export the full catalog (not just filtered) as CSV using **Εξαγωγή Καταλόγου (CSV)**.

---

## Audit Log (`audit.php`)

Admin-only. Shows every significant system action with timestamp, user, action type, target, and details.

Filter by action type, free-text search, or adjust records per page. Export all records as CSV.

The **Backup** button is currently non-functional and shows a warning modal when clicked.

---

## Profile Settings (`profile.php`)

Available to all logged-in employees. Change your own password (requires current password). Recent activity (last 10 audit entries) is shown on the right.

---

## Public Catalog (`catalog_public.php`)

No login required. Visitors can browse, filter, sort, and paginate the collection of items marked as `is_public = 1`.

Filter options: search, category, language, type, publisher. Sort by title (A-Z / Z-A), year (newest/oldest first), or author.

Clicking a row opens a detail modal populated from the row's `data-*` attributes (no AJAX call needed).

The interface supports two languages (Greek / English) switchable client-side. The language preference is saved in `localStorage`.

Staff can log in directly from the public catalog via the **Διαχείριση** button in the top bar, which opens the same login modal used on `index.php`.

---

## Help & FAQ (`help.php`)

A built-in static FAQ page available to all logged-in employees. Accessible from the sidebar under **Βοήθεια & FAQ**.
