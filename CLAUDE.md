# CLAUDE.md — Case Central

This file provides guidance to AI assistants working in this repository.

## Project Overview

**Case Central** is a legal practice management application built on top of [Frappe Framework](https://frappeframework.com/) (the same framework that powers ERPNext). It extends ERPNext with legal-domain-specific doctypes for managing matters, cases, tasks, appointments, legal services, billing, documents, and library books.

- **Publisher**: 4C Solutions
- **License**: MIT
- **Version**: 0.0.1
- **Frappe App Name**: `casecentral`

---

## Repository Structure

```
casecentral/
├── casecentral/                    # Main Python package
│   ├── hooks.py                    # Central Frappe app configuration
│   ├── modules.txt                 # Module declarations
│   ├── patches.txt                 # Migration patch declarations
│   ├── case_central/               # Primary module (33 doctypes)
│   │   ├── doctype/                # Doctype definitions (Python + JSON + JS)
│   │   ├── report/                 # Report definitions
│   │   ├── dashboard_chart/        # Dashboard chart configs
│   │   ├── number_card/            # KPI number cards
│   │   └── workspace/              # Workspace layout
│   ├── legal_documents/            # Legal documents module (3 doctypes)
│   │   └── doctype/
│   ├── config/                     # App config (desktop icons, etc.)
│   ├── doc_events/                 # Document event handlers
│   ├── fixtures/                   # Exported fixtures (custom fields, property setters, client scripts)
│   ├── patches/                    # One-time migration scripts
│   │   └── v14_0/
│   ├── public/                     # Frontend static assets
│   │   ├── js/                     # Custom JS for standard doctypes
│   │   └── casecentral.bundle.js   # JS bundle entry point
│   ├── templates/                  # Jinja HTML templates
│   ├── utils.py                    # Shared utility functions
│   └── www/                        # Web pages and REST endpoints
├── setup.py                        # Python package metadata
├── requirements.txt                # Python dependencies
├── MANIFEST.in
└── .github/workflows/ci.yml        # GitHub Actions CI
```

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Backend framework | Frappe (Python) |
| Python version | 3.10 |
| Database | MariaDB 10.6 |
| Frontend | Frappe Desk (JS + Frappe UI) |
| Node version | 14 |
| Package manager | pip + yarn/npm |
| CI | GitHub Actions |

**Key Python dependency**: `docxtpl` — used to generate `.docx` documents from Jinja2 templates.

---

## Modules and Doctypes

### Case Central Module (33 doctypes)

| Category | Doctypes |
|----------|---------|
| Core | Matter, Case, Task, Task Type, Matter Type, Party |
| Legal Services | Legal Service, Legal Service Entry, Legal Service Rate, Service, Service Type |
| Appointments | Customer Appointment, Appointment Type, Lawyer Schedule, Lawyer Schedule Time Slot |
| Case Details | Case History, Case Party, Interim Applications, Witness Examined, Exhibits |
| Documents | Document Outward, Court Form, Caveat, File Type |
| Library | Book, Book Type, Lend Book |
| Settings | Case Central Settings, Nature of Case, Nature of Disposal, Linked Matter, Meeting Room, Meeting Room Schedule |

### Legal Documents Module (3 doctypes)

- Legal Notice
- Legal Templates
- Postal Order

---

## Key Configuration: hooks.py

`casecentral/hooks.py` is the central configuration file for the Frappe app. It declares:

- **`fixtures`**: Exports `Custom Field`, `Property Setter`, and `Client Script` records — these are stored as JSON in `casecentral/fixtures/` and loaded during app installation.
- **`override_doctype_dashboards`**: Custom dashboard overrides for standard ERPNext doctypes.
- **`doctype_js`**: Associates custom JS files from `public/js/` with standard Frappe doctypes (e.g., `sales_invoice.js`, `payment_entry.js`).
- **`doc_events`**: Maps document lifecycle hooks (on_submit, on_cancel, etc.) to handler functions in `casecentral/doc_events/`.
- **`scheduler_events`**: Defines daily cron jobs (caveat expiration, appointment status updates).
- **`override_doctype_class`**: Replaces Frappe's standard doctype Python classes with custom ones.

---

## Frontend Conventions

Custom JavaScript for **standard ERPNext doctypes** lives in `casecentral/public/js/`:
- `sales_invoice.js` — adds a "Add Legal Services" button and dialog
- `payment_entry.js` — payment entry customizations
- `task_quick_entry.js` — quick entry form for tasks (also imported in the bundle)

These files are referenced in `hooks.py` under `doctype_js`.

Custom JavaScript for **this app's own doctypes** lives alongside the doctype definition, e.g.:
- `casecentral/case_central/doctype/matter/matter.js`

The main JS bundle entry point is `casecentral/public/casecentral.bundle.js`.

---

## API / Whitelisted Methods

Frappe uses a whitelist pattern for Python methods callable from the frontend. Methods decorated with `@frappe.whitelist()` are accessible at:

```
/api/method/<python.dotted.path>
```

Key whitelisted methods:

| Method | Location |
|--------|----------|
| `get_billing_info` | `matter.py` |
| `fetch_book_details(isbn)` | `book.py` — calls Google Books API |
| `set_legal_services(checked_values)` | `sales_invoice.py` override |
| `get_legal_services_to_invoice(matter, company)` | `utils.py` |
| Matter analytics | `report/matter_analytics/` |
| Case Central metrics | `report/case_central_metrics/` |

---

## Fixtures

Fixtures in `casecentral/fixtures/` are JSON exports of Frappe configuration records:

- `custom_field.json` — 150+ custom fields added to standard ERPNext doctypes
- `property_setter.json` — 50+ property overrides (required, hidden, options, etc.)
- `client_script.json` — 6 client-side validation/UI scripts

**Important**: Never edit these JSON files manually. Instead:
1. Make changes through the Frappe Desk UI
2. Run `bench --site <site> export-fixtures` to regenerate the JSON
3. Commit the updated JSON

---

## Database Migrations (Patches)

One-time data migration scripts live in `casecentral/patches/v14_0/`:

- `migrate_mobile_no_and_email_id.py` — migrates contact info to Contact doctype
- `update_matter_type_in_task.py` — backfills matter type on existing tasks

**Adding a new patch:**
1. Create a Python file in the appropriate `patches/` subdirectory
2. Register it in `casecentral/patches.txt`
3. Patches run automatically on `bench --site <site> migrate`

---

## Scheduled Tasks

Defined in `hooks.py` under `scheduler_events`:

| Frequency | Task |
|-----------|------|
| Daily | `caveat.set_expired_status` — mark overdue caveats as expired |
| Daily | `customer_appointment.update_appointment_status` — auto-update appointment statuses |

---

## Development Setup

### Prerequisites

- Python 3.10+
- Node.js 14
- MariaDB 10.6
- Redis
- `frappe-bench` CLI

### Initial Setup

```bash
# Install bench
pip install frappe-bench

# Initialize a new bench (downloads Frappe)
bench init --skip-redis-config-generation ~/frappe-bench
cd ~/frappe-bench

# Get this app
bench get-app casecentral /path/to/repo
# or
bench get-app casecentral https://github.com/4csolutions/casecentral

# Install Python dependencies
bench setup requirements --dev

# Create a new site
bench new-site --mariadb-root-password <root-pass> mysite.localhost

# Install the app on the site
bench --site mysite.localhost install-app casecentral

# Build frontend assets
bench build

# Start the development server
bench start
```

### Environment / Site Configuration

Frappe stores site-level configuration (DB credentials, etc.) in `sites/<site>/site_config.json`. There is no `.env` file — configuration is managed through Frappe's site config system.

---

## Running Tests

```bash
# Enable tests on the site
bench --site <site> set-config allow_tests true

# Run all tests for this app
bench --site <site> run-tests --app casecentral

# Run tests for a specific doctype
bench --site <site> run-tests --doctype "Matter"

# Run a specific test file
bench --site <site> run-tests --module casecentral.case_central.doctype.matter.test_matter
```

Test files follow the naming convention `test_<doctype_name>.py` and are located inside each doctype's directory.

---

## CI/CD

GitHub Actions workflow: `.github/workflows/ci.yml`

**Triggers**: Push to `develop` branch, all pull requests.

**Steps**:
1. Spin up Ubuntu + Python 3.10 + Node 14 + MariaDB 10.6
2. Cache pip/yarn dependencies
3. Install `frappe-bench`, initialize bench, configure MariaDB charset (`utf8mb4`)
4. Clone this app into the bench, install requirements
5. Create test site (`admin`/`admin`), install app, build assets
6. Enable tests and run `bench run-tests --app casecentral`

---

## Coding Conventions

### Python

- Follow Frappe's doctype controller pattern: each doctype has a Python class in `<doctype_name>.py` that extends `frappe.model.document.Document`.
- Use `frappe.db.get_value`, `frappe.get_doc`, `frappe.get_all`, `frappe.db.sql` for database operations — never raw SQL unless necessary.
- Whitelist methods that need to be callable from the frontend with `@frappe.whitelist()`.
- Use `frappe.throw(msg)` for user-facing errors and `frappe.log_error()` for logging exceptions.
- Use `frappe._("string")` for all user-visible strings (i18n).

### JavaScript

- Follow Frappe's `frappe.ui.form.on("DocType", { ... })` pattern for form-level customizations.
- Use `frappe.call({ method: "...", args: {...}, callback: fn })` for backend calls.
- Use `frappe.msgprint` / `frappe.throw` for user feedback.

### Fixtures

- Do not hand-edit fixture JSON files.
- Always export fixtures via `bench export-fixtures` after UI changes.

### Patches

- Each patch must be idempotent (safe to run multiple times).
- Check for existing data before modifying.

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `casecentral/hooks.py` | Central app configuration |
| `casecentral/modules.txt` | Module list |
| `casecentral/patches.txt` | Patch registry |
| `casecentral/utils.py` | Shared helper functions |
| `casecentral/fixtures/custom_field.json` | Custom field definitions |
| `casecentral/fixtures/client_script.json` | Client scripts |
| `casecentral/public/js/sales_invoice.js` | Sales Invoice UI extension |
| `setup.py` | Python package config |
| `requirements.txt` | Python deps (`docxtpl`) |
| `.github/workflows/ci.yml` | CI pipeline |
