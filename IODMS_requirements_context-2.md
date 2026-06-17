# IODMS – Project Context & Requirements
## Inward/Outward Document Management System
### HAL AURDC (Aircraft Upgrade Research & Design Centre), Nashik – DEA (Design & Engineering Activity)

---

## Project Context

- Redevelopment of a legacy Microsoft Access-based document management system
- Legacy DB is inaccessible; 3 tables exported to XLSX are available
- ~5000+ scanned docs/PDFs from 2006 onwards need to be migrated
- Air-gapped LAN setup (no internet) — HAL is protected under Official Secrets Act 1923
- Central server PC hosts the DB; all clients connect over LAN
- ~30 registered users, max 10 concurrent (very rare peak)
- Stack: React (frontend) + FastAPI (backend, Python) + PostgreSQL (database)
- Frontend: browser-based, Chromium on Windows, no client install required
- On-demand desktop app packaging (e.g. Electron) from same codebase
- SDLC: V-Model (4-phase). Current phase: Requirements (SRS) complete
- RBAC: two roles — User, Admin

---

## Database Tables (beyond 3 legacy tables)

All dropdown/admin-editable data lives in the DB, not in code.

| Table | Purpose |
|---|---|
| `inward_register` | Legacy + new inward records |
| `outward_register` | Legacy + new outward records |
| `address_book` | Legacy + new address entries |
| `users` | Officer accounts, roles, credentials |
| `designations` | Dropdown master list |
| `organisations` | Dropdown master list |
| `folder_types` | Dropdown master list |
| `address_groups` | Categories used in Create File |
| `draft_files` | Temp outward drafts (Finalise File queue) |
| `audit_log` | All user actions with timestamp |

---

## File & Folder Structure

```
I/O Register/
├── Drafts/              ← temp files before finalisation
├── Outward/
│   └── {Year}/
│       └── {File Type e.g. Su-30, LCA}/
└── Inward/
    └── {Year}/
        └── {File Type}/
```

- File numbering scheme: TBD (stakeholder to confirm), yearly reset required
- File folder names correspond to file numbers selected in Create File

---

## Modules & Functional Requirements

### Module 1 — Create File (Outward Document Creation)

| FR ID | Requirement |
|---|---|
| FR-001 | Text input for document Subject |
| FR-002 | Dropdown: Created By (registered user list) |
| FR-003 | Date picker: Created On (default today) |
| FR-004 | Dropdown: File Number; auto-displays File Folder Name on selection |
| FR-005 | Dropdown: Category (address group from address book) |
| FR-006 | Address To list — dynamically populated from selected category |
| FR-007 | Display area for selected primary addresses |
| FR-008 | CC — multi-select from address book dropdown |
| FR-009 | Template selection: Fax/Outside (with GM Sig), Fax/Outside (without GM Sig), Internal Letter |
| FR-010 | Generate pre-filled .docx draft from template + form data; user can edit before saving |
| FR-011 | On save: draft stored to Drafts folder; filename format `fax-{user}-{date}-{time}.docx`; entry created in draft_files table |

---

### Module 2 — Finalise File (Draft Queue)

| FR ID | Requirement |
|---|---|
| FR-012 | Table view of all drafts: Link Address, Register No., Issuing Date, To, Subject, Remarks, Created By, Created On (date+time) |
| FR-013 | Finalise File action: auto-generate Outward Register Number + Date of Finalisation; move record to Outward Register |
| FR-014 | Move to Trash action: soft delete draft |
| FR-015 | On finalisation: file moved from Drafts/ to Outward/{Year}/{File Type}/ |

---

### Module 3 — Inward Entry

| FR ID | Requirement |
|---|---|
| FR-016 | Dept Inward No. — auto-generated on New |
| FR-017 | Date of Inward — date picker (default today) |
| FR-018 | Scanned Format — dropdown; plus option to trigger connected scanner directly (SDK/specs TBD) |
| FR-019 | Letter Ref No. — text input |
| FR-020 | Dated — date of original letter |
| FR-021 | Received From — dropdown from address book |
| FR-022 | Originated By — dropdown from address book |
| FR-023 | Subject — text input |
| FR-024 | Referred To — dropdown/text |
| FR-025 | Folder Type — dropdown from address book |
| FR-026 | CC Sent To — multi-select |
| FR-027 | Remarks — text |
| FR-028 | Type — dropdown: Query / Snag / File |
| FR-029 | Status — toggle: A (Active) / NA (Not Active) |
| FR-030 | Operations: New, Modify, Save |

---

### Module 4 — Inward Register

| FR ID | Requirement |
|---|---|
| FR-031 | Table: Receiving Date, Inward Number, Inward Letter No., Inward Date, Received From, Subject, Originated By, Referred To, File No., CC Sent To, Remarks, Type, Status |
| FR-032 | Year filter — dropdown |
| FR-033 | Search by: Referred To, Received From, Originated By, Subject |

---

### Module 5 — Outward Register

| FR ID | Requirement |
|---|---|
| FR-034 | Table: Register No., File No., Issuing Date, To, Subject, Remarks, Created By |
| FR-035 | Year filter — dropdown |
| FR-036 | Search by: File Number (dropdown), Created By (dropdown), Address To (text), Subject (text) |

---

### Module 6 — Address Book

| FR ID | Requirement |
|---|---|
| FR-037 | Table: Address ID, Name, Designation, Organisation, Address 1, Address 2, Fax No., Email, Type |
| FR-038 | Search by: Name, Designation, Organisation, Fax No., Email (dropdown selector + text input) |
| FR-039 | Add / Modify form: Address ID (auto-gen), Name, Address (2 lines), Designation (dropdown), Organisation (dropdown), Fax No., Email, Type (default: Others) |
| FR-040 | Address ID auto-generated on new entry |
| FR-041 | Designation and Organisation dropdowns sourced from master lists in Admin Panel |
| FR-042 | Address groups/categories managed via Admin Panel; used in Create File Category dropdown |

---

### Module 7 — Admin Panel

| FR ID | Requirement |
|---|---|
| FR-043 | Create new officer/user: name, designation (dropdown), department, login credentials |
| FR-044 | Activate / Deactivate existing user accounts |
| FR-045 | Password reset for any user |
| FR-046 | [PLACEHOLDER] Audit log viewer — TBD in HLD |
| FR-047 | [PLACEHOLDER] Database backup trigger — TBD in HLD |
| FR-048 | [PLACEHOLDER] Yearly register number reset — TBD in HLD |
| FR-049 | [PLACEHOLDER] Master list management: Designations, Organisations, Folder Types, Address Groups — TBD in HLD |
| FR-050 | RBAC enforcement: Admin accesses all modules; User restricted to non-admin modules |

---

### Module 8 — Authentication

| FR ID | Requirement |
|---|---|
| FR-051 | Login screen with role selection (User/Admin), username, password |
| FR-052 | Password reset — user-initiated or admin-triggered |
| FR-053 | Session timeout after configurable idle period (default 30 min) |
| FR-054 | No SSO, no OTP — credential-based auth only |

---

## Non-Functional Requirements (Summary)

| NFR ID | Category | Requirement |
|---|---|---|
| NFR-001 | Security | Air-gapped LAN only; no internet calls; OSA 1923 compliant |
| NFR-002 | Security | Passwords stored as bcrypt hashes |
| NFR-003 | Security | Session timeout (default 30 min idle) |
| NFR-004 | Security | Full audit trail: user + action + timestamp on all record changes |
| NFR-005 | Performance | <3s query response for 50,000+ records |
| NFR-006 | Performance | Support 30 users, 10 concurrent |
| NFR-007 | Reliability | Daily DB backup; 99% uptime during office hours |
| NFR-008 | Usability | Chromium browser on Windows; no client install |
| NFR-009 | Maintainability | Modular, heavily commented code; maintainable by single developer |
| NFR-010 | Scalability | Schema supports 10+ years of records |
| NFR-011 | Compatibility | Packagable as desktop app on demand (Electron or equivalent) |
| NFR-012 | Migration | One-time migration from 3 legacy XLSX tables to PostgreSQL |

---

## TBD / Open Items

- FR-041 / File numbering scheme — confirm with stakeholder; yearly reset confirmed
- FR-018 / Scanner SDK and driver specs
- FR-046 to FR-049 / Admin Panel detailed requirements
- Desktop packaging tool (Electron or equivalent)
