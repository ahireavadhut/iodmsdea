# IODMS – High Level Design (HLD)
### HAL AURDC Nashik — DEA | Standalone PC Phase (Phase 2A)

---

## 1. Purpose of This Document

This document describes the architecture of the Inward/Outward Document Management System (IODMS) at the component level. It answers:

- What are the system's 3 main components and what does each one do?
- How do they connect and talk to each other?
- For each of the 3 main workflows, how does data move from the user's click to the database and back?

This is the design baseline for Phase 2A (Standalone PC). The LAN network configuration (Phase 2C) will layer on top of this design without changing it structurally.

---

## 2. System Components

```
┌─────────────────────────────────────────────────────────┐
│                    User's Browser                        │
│              React Frontend (Chromium)                   │
│   Screens: Login, Create File, Inward Entry,            │
│   Registers, Address Book, Admin Panel                   │
└────────────────────┬────────────────────────────────────┘
                     │  HTTP requests (JSON)
                     │  e.g. GET /inward, POST /file/create
                     ▼
┌─────────────────────────────────────────────────────────┐
│                  FastAPI Backend                          │
│              Python, runs on localhost:8000              │
│  - Receives all requests from the browser               │
│  - Validates inputs and enforces business rules         │
│  - Reads/writes to PostgreSQL                           │
│  - Generates .docx files from templates                 │
│  - Moves files between folders on disk                  │
│  - Logs every action to audit_log                       │
└──────────┬──────────────────────────┬───────────────────┘
           │  SQL queries             │  File I/O
           ▼                          ▼
┌──────────────────┐      ┌──────────────────────────────┐
│   PostgreSQL DB  │      │       File System             │
│  localhost:5432  │      │  I/O Register/               │
│                  │      │  ├── Drafts/                  │
│  All records,    │      │  ├── Outward/{Year}/{Type}/   │
│  users, master   │      │  └── Inward/{Year}/{Type}/   │
│  lists, drafts,  │      │                              │
│  audit trail     │      │  .docx files live here       │
└──────────────────┘      └──────────────────────────────┘
```

---

### 2.1 Browser — React Frontend

**What it is:** The interface the user sees and operates in. Runs in Chromium on Windows. No installation required on the client machine — the user just opens a URL.

**What it does:**
- Renders every screen: Login, Create File, Finalise File (draft queue), Inward Entry, Inward Register, Outward Register, Address Book, Admin Panel.
- Sends HTTP requests to FastAPI whenever the user clicks a button, submits a form, or loads a page.
- Receives JSON data from FastAPI and displays it in tables, dropdowns, and form fields.
- Holds no business logic. It does not talk to the database directly. It does not move files. All of that happens in FastAPI.

**Session handling:** On login, FastAPI returns a session token. React stores the token in memory and sends it with every subsequent request. If the user is idle for 30 minutes, the session expires and the user is returned to the login screen (FR-053).

---

### 2.2 FastAPI — Backend / Middleman

**What it is:** A Python web server running on the same PC (standalone phase: `localhost:8000`). Every action the user takes in the browser becomes an HTTP request that arrives here.

**What it does:**

| Responsibility | Detail |
|---|---|
| Request routing | Each URL maps to a Python function (e.g. `POST /inward/new` → `create_inward_entry()`) |
| Input validation | Checks required fields, data types, and value ranges before touching the database |
| Business logic | Auto-generates Inward Numbers (FR-016), Outward Register Numbers (FR-013), Address IDs (FR-040), draft filenames (FR-011) |
| Database access | Reads and writes all records via SQL queries to PostgreSQL |
| Document generation | Fills a .docx template with form data and writes the output to `Drafts/` (FR-010) |
| File movement | On finalisation, moves the .docx from `Drafts/` to `Outward/{Year}/{File Type}/` (FR-015) |
| Authentication | Verifies credentials at login, issues session tokens, enforces role-based access (FR-050) |
| Audit logging | Writes a row to `audit_log` after every create, modify, finalise, or delete action (NFR-004) |

**Routers (one per module):**

```
backend/
├── main.py                  ← starts the server, registers all routers
├── database.py              ← PostgreSQL connection
├── routers/
│   ├── auth.py              ← login, session, password reset
│   ├── create_file.py       ← Create File workflow
│   ├── finalise_file.py     ← Finalise File / draft queue
│   ├── inward.py            ← Inward Entry + Inward Register
│   ├── outward.py           ← Outward Register (read-only view)
│   ├── address_book.py      ← Address Book CRUD
│   └── admin.py             ← user management, master lists, audit log
└── models/
    ├── inward.py            ← inward_register table model
    ├── outward.py           ← outward_register + draft_files models
    ├── address.py           ← address_book + address_groups models
    └── user.py              ← users + audit_log models
```

---

### 2.3 PostgreSQL — Database

**What it is:** The persistent data store. All records that need to survive a restart — every inward entry, every outward record, every address, every user account, every dropdown option — live here.

**What it does not do:** It does not store .docx files. Files live on the file system. The database stores metadata about files (path, draft status, register number).

**Tables and their roles in the system:**

| Table | Populated by | Read by |
|---|---|---|
| `users` | Admin Panel | Login screen, Created By dropdown, all modules |
| `designations` | Admin Panel | Address Book form, user creation form |
| `organisations` | Admin Panel | Address Book form |
| `folder_types` | Admin Panel | Inward Entry (FR-025), Create File (FR-004) |
| `address_groups` | Admin Panel | Create File Category dropdown (FR-005) |
| `address_book` | Address Book module | Create File address list, Inward Entry dropdowns |
| `inward_register` | Inward Entry | Inward Register view |
| `draft_files` | Create File (on save) | Finalise File queue |
| `outward_register` | Finalise File (on finalise) | Outward Register view |
| `audit_log` | FastAPI (automatic, every write) | Admin Panel audit viewer |

---

### 2.4 File System — .docx Storage

**What it is:** A folder hierarchy on the PC's hard disk. No database inside — just folders and files.

**Folder structure:**

```
I/O Register/
├── Drafts/                        ← .docx files saved during Create File
│                                     filename: fax-{user}-{date}-{time}.docx
├── Outward/
│   └── {Year}/
│       └── {File Type}/           ← .docx moved here on Finalise
│           └── {file}.docx
└── Inward/
    └── {Year}/
        └── {File Type}/           ← scanned PDFs/docs stored here
            └── {scan}.pdf
```

**How FastAPI interacts with it:**
- On Create File save: FastAPI writes a new `.docx` to `Drafts/` using the `python-docx` library.
- On Finalise File: FastAPI moves the file from `Drafts/` to `Outward/{Year}/{File Type}/` using a standard Python file move.
- On Inward Entry: The path of the scanned document is stored in `inward_register.scan_path`. The file itself is already in `Inward/{Year}/{File Type}/` (placed there manually or via scanner integration).

---

## 3. Data Flow — Three Main Workflows

---

### Workflow 1 — Create Outward File

**Plain English:** An officer fills in a form, picks a template, and the system generates a pre-filled Word document for them to edit and save.

```
USER (Browser)                  FastAPI                    PostgreSQL / File System
─────────────────────────────────────────────────────────────────────────────────
1. Opens Create File screen
                         ──GET /dropdowns──►  reads: folder_types, address_groups,
                         ◄──JSON────────────  address_book, users → returns to browser

2. Fills in: Subject, Created By,
   Created On, File Number,
   Category, Address To,
   CC, Template type

3. Clicks "Generate Document"
                         ──POST /file/create──►
                                               validates all fields
                                               reads selected address rows from address_book
                                               fills .docx template with form data
                                               writes .docx to Drafts/
                                                 filename: fax-{user}-{date}-{time}.docx
                                               inserts row into draft_files
                                                 (subject, to, cc, file_no, draft_path, created_by, created_on)
                                               writes row to audit_log
                         ◄──{draft_id, file_path, preview_url}──

4. Browser opens the .docx
   preview; user edits if needed.
   No further DB action at this stage.
```

**What ends up where after this workflow:**
- `draft_files` table: 1 new row with `status = 'draft'`
- `Drafts/` folder: 1 new `.docx` file
- `audit_log`: 1 new row — user, action `CREATE_DRAFT`, timestamp

---

### Workflow 2 — Finalise File (Draft → Outward Register)

**Plain English:** An officer reviews the saved draft and clicks Finalise. The system assigns a permanent register number, moves the file to the archive folder, and records it in the Outward Register.

```
USER (Browser)                  FastAPI                    PostgreSQL / File System
─────────────────────────────────────────────────────────────────────────────────
1. Opens Finalise File screen
                         ──GET /drafts──►     reads all rows from draft_files
                         ◄──JSON────────────  where status = 'draft'
   Table displayed: Link Address,
   Register No. (blank), Issuing Date,
   To, Subject, Remarks,
   Created By, Created On

2. Clicks "Finalise" on one row
                         ──POST /file/finalise/{draft_id}──►
                                               generates next Outward Register No.
                                                 (sequential, yearly reset — FR-013)
                                               sets Issuing Date = today
                                               inserts row into outward_register
                                               updates draft_files: status = 'finalised'
                                               moves .docx from:
                                                 Drafts/{filename}.docx
                                                 → Outward/{Year}/{File Type}/{filename}.docx
                                               writes row to audit_log
                         ◄──{register_no, outward_path}──

3. Browser refreshes draft queue.
   Finalised row no longer appears.
   User can view it in Outward Register.

3b. [Alternate] Clicks "Move to Trash"
                         ──POST /file/trash/{draft_id}──►
                                               updates draft_files: status = 'trashed'
                                               writes row to audit_log
                         ◄──{success}──
```

**What ends up where after this workflow:**
- `outward_register` table: 1 new permanent record
- `draft_files` table: row updated to `status = 'finalised'`
- `Outward/{Year}/{File Type}/` folder: `.docx` file now lives here
- `Drafts/` folder: file removed
- `audit_log`: 1 new row — user, action `FINALISE_FILE`, register number, timestamp

---

### Workflow 3 — Inward Entry

**Plain English:** A physical document (letter, fax, query) has been received and scanned. An officer creates a record for it in the system.

```
USER (Browser)                  FastAPI                    PostgreSQL / File System
─────────────────────────────────────────────────────────────────────────────────
1. Opens Inward Entry screen,
   clicks "New"
                         ──POST /inward/new──►
                                               generates next Dept Inward No.
                                                 (sequential, yearly reset — FR-016)
                                               returns blank form with auto-filled number
                         ◄──{inward_no, today's date}──

2. Fills in all fields:
   Date of Inward, Scanned Format,
   Letter Ref No., Dated,
   Received From (dropdown → address_book),
   Originated By (dropdown → address_book),
   Subject, Referred To,
   Folder Type, CC Sent To,
   Remarks, Type, Status

3. Clicks "Save"
                         ──POST /inward/save──►
                                               validates all fields
                                               inserts row into inward_register with:
                                                 inward_no, date_of_inward, scan_format,
                                                 letter_ref_no, letter_date,
                                                 received_from (FK → address_book),
                                                 originated_by (FK → address_book),
                                                 subject, referred_to, folder_type,
                                                 cc_sent_to, remarks, type, status
                                               writes row to audit_log
                         ◄──{success, inward_id}──

4. [Optional] User clicks "Modify"
   on an existing record
                         ──GET /inward/{id}──►  reads row from inward_register
                         ◄──{record JSON}──
   Form re-populates with existing data.
   User edits and clicks Save again.
                         ──PUT /inward/{id}──►  updates row in inward_register
                                               writes row to audit_log (action: MODIFY)
                         ◄──{success}──
```

**What ends up where after this workflow:**
- `inward_register` table: 1 new row (or 1 updated row on Modify)
- File system: the scanned document is already in `Inward/{Year}/{File Type}/` — the record in the DB stores its path; no file movement happens during Inward Entry
- `audit_log`: 1 new row — user, action `CREATE_INWARD` or `MODIFY_INWARD`, timestamp

---

## 4. Cross-Cutting Concerns

### 4.1 Authentication Flow

Every request from the browser (except `POST /login`) must carry a valid session token in the header. FastAPI checks the token before running any logic.

```
Browser ──POST /login {username, password, role}──► FastAPI
         validates credentials against users table (bcrypt hash check)
         if valid: issues session token, stores in server-side session store
◄──{token, role, username}──
Browser stores token in memory (not localStorage), attaches to all future requests.
On 30-min idle: FastAPI invalidates token → browser redirects to login screen.
```

### 4.2 Role-Based Access

| Module | User | Admin |
|---|---|---|
| Create File | ✅ | ✅ |
| Finalise File | ✅ | ✅ |
| Inward Entry | ✅ | ✅ |
| Inward Register | ✅ | ✅ |
| Outward Register | ✅ | ✅ |
| Address Book | ✅ | ✅ |
| Admin Panel | ❌ | ✅ |

FastAPI enforces role on every route. The browser hides restricted UI, but FastAPI is the enforcing layer.

### 4.3 Audit Log

Every `POST`, `PUT`, or `DELETE` action in FastAPI ends with an insert into `audit_log`:

```
audit_log row: user_id | action | target_table | target_id | timestamp
```

The Admin Panel will surface this as a searchable/filterable table (FR-046, detail TBD in schema step).

### 4.4 Dropdown Master Lists

All dropdowns whose options Admin can edit are sourced from the database, not hardcoded:

```
designations → used in: Address Book (Designation field), Admin Panel (user creation)
organisations → used in: Address Book (Organisation field)
folder_types  → used in: Inward Entry (Folder Type), Create File (File Number display)
address_groups → used in: Create File (Category dropdown → drives Address To list)
```

When Admin adds a new option in the Admin Panel, it is a `POST` to the relevant master list table. The next time any form loads, the new option appears in the dropdown.

---

## 5. Standalone PC Assumptions (Phase 2A)

These constraints apply for the current build. They will change in Phase 2C (LAN).

| Item | Standalone value | Will change in Phase 2C? |
|---|---|---|
| FastAPI host | `localhost:8000` | Yes → server LAN IP |
| PostgreSQL host | `localhost:5432` | Yes → allow LAN connections |
| File system path | Local disk path on same PC | Yes → shared or server path |
| Concurrent users | 1 (same PC) | No code change — config only |
| Network | None required | LAN, air-gapped |

---

## 6. Open Items Flagged for Schema Step

The following items from the requirements are TBD and will be resolved in the Database Schema step (Step 2):

| FR ID | Open item |
|---|---|
| FR-041 | File numbering scheme — format and yearly reset logic (stakeholder to confirm) |
| FR-018 | Scanner SDK/driver specs — inward scan trigger integration |
| FR-046 | Audit log viewer — columns, filters, pagination |
| FR-047 | Database backup trigger — manual button or scheduled? |
| FR-048 | Yearly register number reset — automatic on Jan 1 or manual? |
| FR-049 | Master list management UI — inline edit or modal form? |
