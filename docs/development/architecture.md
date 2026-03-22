# Architecture Overview

## Design Philosophy

openGPLMS is a deliberately simple, self-contained PHP application with no framework, no build step, and no package manager. Every PHP file is a standalone page that includes `config.php` for shared functions and the layout files for the shared HTML shell. This makes the codebase easy to read, deploy, and modify without specialized tooling.

---

## Directory Layout

```
src/
├── config.php              ← Single config + all shared PHP functions
├── layout_admin.php        ← HTML <head>, sidebar, main div open
├── layout_admin_end.php    ← main div close, $extraJs, </body></html>
├── assets/
│   ├── styles/             ← Per-page CSS (all loaded by layout_admin.php)
│   ├── favicon.png
│   └── opengplms-logo.png
├── removit/                ← Dev utility (DELETE after use)
│   ├── install.php
│   ├── login.php
│   └── login.css
└── [page files]            ← One PHP file per feature/page
```

---

## Request Lifecycle

Every page follows this pattern:

```
1. require 'config.php'        → DB connection, session start, all helpers
2. requireEmployee() / requireAdmin()  → Auth guard (redirect if unauthorized)
3. $db = getDB()               → PDO singleton
4. if (POST) { ... }           → Handle mutations, end with header(Location) + exit
5. [Query data for the view]
6. $pageTitle = '...'          → Used by layout
7. $activePage = '...'         → Used by sidebar to highlight the active link
8. include 'layout_admin.php'  → Outputs HTML head + sidebar + flash
9. [Page HTML content]
10. $extraJs = '...'           → Optional JS for this page
11. include 'layout_admin_end.php'  → Closes layout, outputs $extraJs
```

---

## Patterns in Use

### Post-Redirect-Get (PRG)

Every POST handler ends with a redirect:
```php
flash('Success message.');
header('Location: current_page.php'); exit;
```
This prevents the browser from re-submitting the form on F5 refresh.

### Singleton Database Connection

`getDB()` in `config.php` uses a `static $pdo = null` variable. The PDO connection is created once per request and reused across all function calls. This avoids multiple connections to MySQL within a single page load.

### Dynamic ENUM Introspection

Rather than hardcoding the item type list in PHP or JavaScript, several pages query MySQL for the ENUM definition:

```php
$typeResult = $db->query("SHOW COLUMNS FROM {prefix}books LIKE 'type'")->fetch();
preg_match("/^enum\((.*)\)$/", $typeResult['Type'], $matches);
$allTypes = array_map(fn($v) => trim($v, "'"), explode(",", $matches[1]));
```

This means adding a new item type only requires an `ALTER TABLE` — no PHP or JS changes needed.

### SearchableSelect Pattern

Custom dropdown components use a three-part structure:
1. A hidden native `<select>` — holds the actual POST value
2. A visible text input — the user types to filter
3. A floating panel div — shows filtered options

JavaScript keeps the hidden select synchronized with the visible input. This approach provides good UX (type-to-filter) while maintaining form compatibility with graceful degradation (the hidden select works even without JavaScript, though without the filter UI).

### Client-Side Data via PHP JSON

Pages that need JavaScript-accessible data serialize PHP arrays to JSON inline:

```php
$jsCategories = json_encode(array_map(fn($c) => ['id' => $c['id'], 'name' => $c['name']], $categories));
```

Then in the HTML:
```html
<script>const DB = { categories: <?= $jsCategories ?> };</script>
```

This avoids AJAX calls for dropdown data that doesn't change mid-session.

### `data-*` Attribute Modal Population (Public Catalog)

The public catalog avoids per-item AJAX by storing all displayable metadata as `data-*` attributes on each `<tr>`. The modal's `show.bs.modal` event reads these attributes and populates the modal DOM:

```javascript
document.getElementById('bookModal').addEventListener('show.bs.modal', function(e) {
    const d = e.relatedTarget.closest('tr').dataset;
    document.getElementById('bmTitle').textContent = d.title;
    // ...
});
```

---

## Frontend Libraries

All loaded from CDN, no local copies:

| Library | Version | Used For |
|---|---|---|
| Bootstrap CSS | 5.3.0 | Grid, utilities, some components |
| Bootstrap JS bundle | 5.3.0 | Modal, alert dismiss, collapse |
| Bootstrap Icons | 1.11.0 | All icons throughout the UI |
| Chart.js | latest (CDN) | Dashboard and reports charts |
| Google Fonts: Playfair Display | — | Headings in admin panel |
| Google Fonts: Source Sans 3 | — | Body text in admin panel |
| Google Fonts: Cormorant Garamond + Jost | — | Landing page and public catalog |
| Font Awesome | 6.5.0 | Login page only (`index.php`) |

---

## CSS Organization

All CSS files live in `src/assets/styles/`. `layout_admin.php` loads all of them unconditionally:

| File | Scope |
|---|---|
| `layout_admin.css` | Sidebar, main layout, shared admin components |
| `add_book.css` | Book form, SearchableSelect dropdown styles |
| `catalog.css` | Catalog table, badges, mass action bar, modals |
| `categories.css` | Category/publisher panels, mini-filter bar |
| `csv_editor.css` | Spreadsheet editor table and toolbar |
| `messages.css` | Two-column message layout, read pane |
| `users.css` | User table, modal styles |
| `index.css` | Public catalog styles |
| `index-styles.css` | Additional public catalog styles |

The CSS for `index.php` (landing page) is inlined in the `<style>` tag within that file because it's a standalone page that does not use `layout_admin.php`.

---

## No Framework, No Build Step

Deliberately chosen constraints:
- **No Composer** — no autoloading, no vendor directory, no dependency management
- **No npm / webpack** — no transpilation, no bundling
- **No MVC framework** — each PHP file is its own controller and view

This makes the project accessible to contributors who know basic PHP, trivial to deploy (copy files + run installer), and easy to audit (no hidden complexity in framework internals).

The trade-off is some code duplication (e.g. `pageWindow()` is defined in four separate files) and inline CSS/JS in PHP files. These are accepted trade-offs for the target use case: small to medium libraries deploying a self-hosted tool.
