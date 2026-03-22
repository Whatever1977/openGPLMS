# CSV Import & Editor Guide

openGPLMS provides two ways to bulk-import books: a file upload importer and a browser-based spreadsheet editor.

---

## CSV Column Format

Columns must appear in this exact order (the header row is optional but recommended):

| # | Column | Required | Notes |
|---|---|---|---|
| 1 | `title` | ✓ | Book title |
| 2 | `author` | ✓ | Author or creator |
| 3 | `isbn` | | ISBN-13 or ISSN; leave blank if none |
| 4 | `type` | | One of the ENUM values (see below); defaults to `Βιβλίο` |
| 5 | `category` | | Category name; created automatically if missing |
| 6 | `publisher` | | Publisher name; created automatically if missing |
| 7 | `year` | | Publication year (integer) |
| 8 | `language` | | Free text; defaults to `Ελληνικά` |
| 9 | `pages` | | Integer |
| 10 | `edition` | | Free text (e.g. `Α΄`) |
| 11 | `volume` | | Free text (e.g. `Vol.1`) |
| 12 | `location` | | Shelf location (e.g. `Α1-Σ2`) |
| 13 | `description` | | Notes or abstract |
| 14 | `status` | | `Διαθέσιμο` / `Μη Διαθέσιμο` / `Σε Επεξεργασία`; defaults to `Διαθέσιμο` |
| 15 | `is_public` | | `1` = public (default), `0` = private |
| 16 | `cover_url` | | Full `https://` URL to cover image |

### Valid Type Values

These must match the database ENUM exactly (case-sensitive):

- `Βιβλίο`
- `Περιοδικό`
- `Εφημερίδα`
- `Χειρόγραφο`
- `Ημερολόγιο`
- `Επιστολή`
- `Άλλο`

Any unrecognized type falls back to `Βιβλίο`.

---

## Downloading the Template

From the **Εισαγωγή CSV** page, click **Κατέβασε Template**. This downloads a UTF-8 BOM CSV file with the correct headers and two example rows.

---

## Uploading a CSV (`csv_import.php`)

1. Navigate to **Εισαγωγή CSV** in the sidebar
2. Drag and drop your file onto the drop zone, or click to select it
3. Choose the delimiter (`,` comma, `;` semicolon, or Tab)
4. Check **Παράλειψη 1ης γραμμής** if your file has a header row (recommended)
5. Check **Ενημέρωση αν υπάρχει ISBN** if you want existing records to be updated when an ISBN match is found; otherwise duplicates are skipped
6. Click **Εισαγωγή Δεδομένων**

### What the Importer Does

- Strips the UTF-8 BOM if present
- Skips empty rows
- Validates that `title` and `author` are non-empty; errors these rows and continues
- Calls `findOrCreate()` for category and publisher names — if a name does not exist in the database it is inserted automatically
- Checks for ISBN duplicates: skips or updates depending on the **Ενημέρωση** checkbox
- Logs the import action (`csv_import`) in the audit log with counts of imported and skipped rows

### Results Table

After import, a results table shows each row with a status badge:
- **Εισήχθη** (green) — new record created
- **Ενημερώθηκε** (amber) — existing record updated
- **Παραλείφθηκε** (red) — skipped (duplicate ISBN or validation error)

---

## Online CSV Editor (`csv_editor.php`)

The editor is a browser-based spreadsheet that lets you create or edit CSV data before importing it, without needing Excel or another application.

### Loading an Existing File

Click **Φόρτωση** and select a `.csv` file from your computer. The editor auto-detects the delimiter (`,`, `;`, or Tab) and parses the content into the table. A header row is skipped automatically if it contains `title` or `τίτλ`.

### Editing

Each column maps directly to a CSV field. Cells use:
- `<select>` for fixed-value fields (type, status, is_public)
- Text input with `<datalist>` autocomplete for free-text-with-suggestions fields (category, publisher, language) — you can type any value even if it's not in the list
- Plain text / number inputs for everything else

### Keyboard Shortcuts

| Key | Action |
|---|---|
| `Enter` | Move to the same column in the next row (creates a new row if on the last) |
| `Tab` | Move to the next cell |
| `Ctrl+D` | Fill Down (copies current cell value to all selected rows below, or all rows below if none selected) |

### Toolbar Actions

| Button | Action |
|---|---|
| **Γραμμή** | Add one new empty row |
| **Διαγραφή** | Delete all selected rows |
| **Αντιγραφή γραμμής** | Duplicate the last selected row |
| **Καθαρισμός** | Clear all fields in selected rows |
| **↑ / ↓** | Move a single selected row up or down |
| **Fill Down** | Copy current cell value downward |
| **+5 / +10 / +20 / +50 γραμμές** | Add multiple blank rows at once |
| **Έλεγχος** | Validate all rows (checks required fields) |
| **Κατέβασε CSV** | Download the table as a `.csv` file |
| **Απευθείας Εισαγωγή** | Generate the CSV in memory and submit it directly to `csv_import.php` without saving locally |

### Important: Checkbox Exclusion Fix

The editor explicitly excludes the row-selection checkbox when collecting cell values using `getRowInputs()`. This is a critical implementation detail — without it, the checkbox `"on"` value would appear as the first column (title) in the exported CSV.

---

## Tips

- Categories and publishers are auto-created during import. You don't need to pre-populate them.
- The `cover_url` field must be a full `https://` URL. The importer does not sanitize it — the sanitization happens only through the web form (`add_book.php`). Validate URLs before importing.
- For files with irregular encodings, open in a text editor and save as UTF-8 (with or without BOM) before importing.
- The importer reads a maximum of 2000 bytes per row (`fgetcsv($handle, 2000, $delimiter)`). Rows with very long descriptions may need to be trimmed before import.
