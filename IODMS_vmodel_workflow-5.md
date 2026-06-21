# IODMS – V-Model Workflow Guide (v4)
## Who this is for: First-year Aviation intern, non-CSE background, learning as you go
## Stack: React + Material UI · FastAPI / Python · PostgreSQL
## AI tooling: Claude Code and/or Google Antigravity (agentic, filesystem-aware coding tools)

> **What changed in this version:** Requirements moved from v1 → v2 (FR-001–054 → FR-000–154, 4 modules → 11 modules, terminology renamed throughout) — that's what v3 of this guide covered. **v4 swaps the working method from "paste documents into a chat box" to using an agentic coding tool (Claude Code or Google Antigravity) that reads and writes your project files directly.** The Two Rules below are unchanged in spirit, but how you *enforce* them changes — see the new section right after this one.

---

## What is the V-Model (Plain English)

Normal software development goes in a straight line: design → build → test.

The V-Model folds that line into a V shape. The left side goes down (design), the bottom is coding, and the right side goes up (testing). The key idea is: **every design decision on the left has a matching test on the right.**

Our V is a branched V — the standalone PC phase comes first, and the LAN network phase branches out of it after testing is done.

```
[Phase 1] Requirements (SRS)
            |
            |
     [Phase 2A] Standalone PC Design
            |
            |
     [Phase 2B] Code — Standalone PC
            |        \
            |         \
     [Phase 3A]    [Phase 2C] LAN Network Design
     Standalone         |
     Testing            |
                   [Phase 3B] Code — LAN Changes
                        |
                   [Phase 3C] LAN Testing
                        |
                   [Deployment]
```

You are currently between Phase 1 (done) and Phase 2A (starting next).

---

## The Two Rules This Whole Guide Follows

1. **Two-file rule.** At any point in the project, AI is fed at most **two** documents: `IODMS_requirements_v2.md` (frozen, never edited) and `IODMS_design.md` (the one evolving technical document — schema, architecture, file list, config, test log all live inside it as sections). No `schema.sql`, no separate `folder_structure.md`, no pile of config files. One doc accumulates everything technical; one doc is the unchanging source of truth.
2. **Minimum file count rule.** The codebase itself is grouped by *module-group*, not by every individual FR or table. One backend router file and one frontend page file per module-group, plus a small shared core. This directly satisfies `NFR-012` ("minimum number of source files... maintainable by a single developer").

Keep both rules in mind through every phase below.

---

## If You're Using Claude Code or Antigravity

Both are **agentic** tools: you point them at a project folder and they read, write, and run files themselves — there's no chat box to paste documents into. This changes *how* the Two Rules get enforced, not the rules themselves.

**1. Set up your project folder first.**
Create an empty folder (e.g. `IODMS/`), put `IODMS_requirements_v2.md` in it, and `git init`. Open that folder as your workspace in Claude Code (`claude` in the terminal, inside the folder) or as a Project in Antigravity (Select Project → New Project → Add Folder).

**2. Replace manual pasting with a persistent context file — this is how the Two-File Rule survives without you re-pasting anything.**

| Tool | File to create at project root | What goes in it |
|---|---|---|
| **Claude Code** | `CLAUDE.md` | Short instructions, not the full documents — see below. Read automatically every session. |
| **Antigravity** | `AGENTS.md` (or `.agents/rules/project-context.md` set to **Always On**) | Same content. Read automatically every session; rules are capped at 12,000 characters so keep it short. |

Put something like this in whichever file applies:

```markdown
# IODMS Project Rules

Before any task in this repo, read in full:
- IODMS_requirements_v2.md (frozen — never edit this file)
- IODMS_design.md (the one evolving technical doc — schema, file tree,
  LAN config, and test log all live here as sections)

Never paste, summarise, or re-derive a third standalone design document.
If new technical decisions are needed, add a new section to
IODMS_design.md — do not create schema.sql, folder_structure.md,
config.md, or any other separate file.

Use only canonical terms from the Terminology Reference table in
IODMS_requirements_v2.md (Folder, Compose Outward, Drafts & Dispatch,
Log Inward, Assign To, Prepared By, etc.) — never legacy names.

The file tree in IODMS_design.md Section C is fixed at 20 files
(10 backend + 10 frontend). Do not create a new file outside that list
without asking first — group new logic into an existing file instead.

Every function/endpoint needs a one-line comment naming the FR ID(s)
it implements.
```

This means you literally never paste `IODMS_requirements_v2.md` or `IODMS_design.md` into a prompt again — the agent re-reads both at the start of every session because `CLAUDE.md`/`AGENTS.md` tells it to. Your job shifts from "remember to paste the right files" to "check the agent actually opened both files" (Claude Code and Antigravity both show a visible "Read file" tool call — glance at it before trusting the output).

**3. Keep a human in the loop, especially early on.**
Both tools default to (or offer) an autopilot mode — Claude Code can be told to auto-approve edits; Antigravity calls this "Agent-driven." Don't use it for this project yet. Use Claude Code's normal per-action confirmation, or Antigravity's **Agent-assisted** or **Review-driven** mode. `NFR-013` says AI must not assume beyond what's written — autopilot makes it much easier to miss a wrong assumption before it's already written to disk.

**4. Don't parallelise the dependency order.**
Antigravity's Manager view can run several agents at once across a project. The Step 2 generation order below is sequential on purpose (`models.py` before `auth.py` before the routers) — running it in parallel risks two agents both deciding to touch `models.py`. If you do try parallel agents, only do it across independent frontend pages that don't share state (e.g. `AddressBook.jsx` alongside `MyProfile.jsx`), never across the backend dependency chain.

**5. Remember the air gap applies to the deployed system, not your laptop.**
Claude Code and Antigravity both call a cloud model over the internet — neither can run on the actual air-gapped server PC. Do all AI-assisted development on an ordinary internet-connected dev machine. `NFR-014`/`EIR-002` (no live internet calls, offline installers via USB) describe the *finished, deployed* IODMS system — not your development environment. Only the built code crosses onto the air-gapped PC, carried over by USB, same as before.

---

## Phase 1 — Requirements ✅ DONE

You interviewed the stakeholders (by looking at the old software), wrote down every screen, every field, every behaviour. That became the SRS.

**Outputs you already have:**
- `IODMS_requirements_v2.md` — FR-000 to FR-154 across 11 modules (Module 0 Auditor View through Module 10 My Profile), full DB schema (11 tables), NFRs, EIRs, RBAC table, terminology reference
- This workflow file

**One thing to never forget:** the requirements doc has a **Terminology Reference** table (Folder, Folder ID, Compose Outward, Drafts & Dispatch, Log Inward, Assign To, Prepared By, etc.). Every prompt should use these canonical terms — never the old legacy names ("Create File", "Finalise File", "Inward Entry", "Referred To"). If the agent ever outputs a legacy term, that's a sign it didn't actually read the requirements doc this session — check that `CLAUDE.md`/`AGENTS.md` is present and ask it to re-read both files before continuing.

---

## Phase 2A — Standalone PC Design

**What standalone means:** The server and the user are on the same computer. No network, no other PCs. You are just making sure the software works correctly before worrying about multiple users on a network.

This phase now has **2 steps**, not 4 — Architecture, Schema, and File Structure are produced together as one document, because they're really one decision (what exists → where it lives → how it's filed), and splitting them into three separate hand-offs was the main source of file pile-up.

---

### Step 1 — Design Document

**① What to feed in**
- `IODMS_requirements_v2.md` — nothing else. No code, no schema, no file tree exist yet. With Claude Code/Antigravity, this just means: the file sits in your project folder and `CLAUDE.md`/`AGENTS.md` is already in place per the previous section — you don't paste it manually.

**② What to do in this step**
AI generates **one** file: `IODMS_design.md`, with three sections.

**Section A — Architecture.** A short HLD describing the 3 system components and how they connect:

```
Browser (React + Material UI)  →  FastAPI (Python)  →  PostgreSQL (Database)
                                        ↕
                                   File System
                            (the .doc drafts/outward/inward folders)
```

For each of the 3 main workflows — **Log Inward**, **Compose Outward → Drafts & Dispatch**, and **Auditor View** — trace the data end to end: what the user clicks, what FastAPI does, what goes into the database, what file moves where.

**Section B — Database Schema.** `CREATE TABLE` statements (with column comments) for all 11 tables, plus a text diagram of foreign keys:

| Table | What it stores |
|---|---|
| `users` | Officer accounts, PB No., DOB, role (User/Admin), bcrypt password hash |
| `folder_types` | Folder ID ↔ Folder Name master list (`Su-30` → `Sukhoi Su-30 MKI`) |
| `address_groups` | Address Group labels |
| `address_book` | All addresses (migrated from legacy XLSX) |
| `received_from_list` | Name-only list for Log Inward "Received From" |
| `originated_by_list` | Name-only list for Log Inward "Originated By" |
| `inward_register` | All inward records (legacy migrated + new) |
| `outward_register` | All dispatched outward records |
| `draft_files` | Draft documents awaiting Dispatch, with `is_locked` / `locked_by` |
| `pending_deletions` | Soft-deleted records awaiting Admin approval (any table) |
| `pending_profile_edits` | User profile change requests awaiting Admin approval |

> ⚠️ There is **no `audit_log` table** — the requirements doc explicitly says one is not required. If AI adds one anyway, it wasn't reading the requirements closely; remove it.

**Section C — File Structure.** A full file tree — every file listed with a one-line comment — grouped by *module-group*, not by individual FR or table:

```
backend/
├── main.py                  # FastAPI app, mounts all routers
├── database.py              # PostgreSQL connection/session
├── models.py                # ALL 11 tables defined here, heavily commented
├── filesystem_utils.py      # shared: next-sequence-number logic, path builder, year rollover
├── auth.py                  # Login, session/RBAC check, Dashboard birthday check, Auditor View
└── routers/
    ├── inward.py            # Module 5 Log Inward + Module 6 Inward Register
    ├── outward.py           # Module 3 Compose Outward + Module 4 Drafts & Dispatch + Module 7 Outward Register
    ├── address_book.py      # Module 8 Address Book
    ├── admin.py             # Module 9, all five sub-tabs (9A–9E)
    └── profile.py           # Module 10 My Profile

frontend/src/
├── App.jsx                  # routing + RBAC route guards
├── api.js                   # all API calls in one place
└── pages/
    ├── AuditorView.jsx      # Module 0
    ├── Login.jsx            # Module 1
    ├── Dashboard.jsx        # Module 2
    ├── Outward.jsx          # Modules 3 + 4 + 7, as internal tabs: Compose / Drafts & Dispatch / Register
    ├── Inward.jsx           # Modules 5 + 6, as internal tabs: Log Inward / Register
    ├── AddressBook.jsx      # Module 8
    ├── AdminPanel.jsx       # Module 9, internal tabs for 9A–9E
    └── MyProfile.jsx        # Module 10
```

That's **10 backend files + 10 frontend files = 20 source files** for the entire application. Compare that to one-file-per-table-and-per-FR-module, which would have been 30+. Long, well-commented files beat many short ones here — that's the whole point of `NFR-012`.

Your job: read all three sections. Cross-check the Architecture section's three workflow traces against what you remember from the old software, cross-check every table column against the FR table, and confirm the file tree has a home for every one of the 11 modules. Flag anything before moving on — this is the only design artifact you'll have, so it needs to be right.

**③ What to feed into AI to execute the next step (Step 2)**
- `IODMS_requirements_v2.md`
- `IODMS_design.md`

That's it — always exactly these two, for the rest of the project.

---

### Step 2 — Code Generation (One Module-Group at a Time)

**① What to feed in — per file**

With `CLAUDE.md`/`AGENTS.md` in place, you don't feed anything manually — the agent re-reads `IODMS_requirements_v2.md` and `IODMS_design.md` itself at the start of the session because the rules file tells it to. Your prompt only needs to name the file to generate and which modules/FRs it covers.

Do **not** paste previously-generated code into the prompt either. `IODMS_design.md` already specifies every table, every function's FR coverage, and every import relationship — that's what it's *for*. If the agent needs to know that `routers/outward.py` imports from `models.py` and `filesystem_utils.py`, it gets that from the file tree comments in `IODMS_design.md` by reading the actual file on disk, not from you re-pasting it.

**② What to do in this step**

Generate code in this dependency order:

1. `database.py` + `models.py` — nothing else can exist without these
2. `filesystem_utils.py` — inward, outward, and drafts all need the same numbering/path logic
3. `auth.py` — login and RBAC; every other router depends on the RBAC check
4. `routers/address_book.py` and `routers/admin.py` — master lists (`folder_types`, `address_groups`, `received_from_list`, `originated_by_list`) must exist before other forms can populate their dropdowns
5. `routers/inward.py`
6. `routers/outward.py`
7. `routers/profile.py`
8. Frontend pages — same order: `App.jsx`/`api.js` → `Login.jsx`/`Dashboard.jsx`/`AuditorView.jsx` → `AddressBook.jsx`/`AdminPanel.jsx` → `Inward.jsx` → `Outward.jsx` → `MyProfile.jsx`

Use this prompt pattern for every file — short, because `CLAUDE.md`/`AGENTS.md` is already carrying the context:

```
Generate backend/routers/outward.py.

Modules this file covers: Module 3 (Compose Outward), Module 4
(Drafts & Dispatch), Module 7 (Outward Register)
Requirements this file covers: FR-030 to FR-057, FR-090 to FR-095

Before writing anything, confirm you've read IODMS_requirements_v2.md
and IODMS_design.md for this session. Add a comment above every
function naming the FR ID it implements. Keep comments simple enough
for a non-CSE beginner to understand. This is one of only 10 backend
files — keep related logic together here rather than creating a new
file.
```

Every function must have an FR comment:

```python
# FR-061: Inward No. is auto-generated, read-only, shown at top of form
@router.post("/inward/new")
def create_inward_entry(...):
    ...
```

This is how you trace every line of code back to a requirement. When something breaks, find the FR, find the function, fix it. Review each file the agent writes (the diff view in Claude Code, or the Artifact/diff in Antigravity) before moving to the next one in the dependency order — don't let it queue up all 10 files unreviewed.

**③ What to feed into AI to execute Phase 3A**
- `IODMS_requirements_v2.md`
- `IODMS_design.md` — the file tree section tells AI which file implements which module, so it knows where to look when writing test cases.

---

## Phase 3A — Standalone Testing

**① What to feed in**
- `IODMS_requirements_v2.md` — the full FR list (FR-000 to FR-154) is the source of truth for what gets tested.
- `IODMS_design.md` — so AI references the right file per test case.

**② What to do in this step**
AI generates a test case document — one row per FR — appended as a new **Section D — Testing Log** inside `IODMS_design.md` (not a separate file). You run each test manually on the standalone PC and mark pass or fail.

| TC ID | FR ID | What you test | What you do | What should happen |
|---|---|---|---|---|
| TC-000 | FR-000 | Auditor View button | Open Login page | "View Registers (Auditor)" button visible below login form |
| TC-061 | FR-061 | Inward No. auto-generate | Click New in Log Inward | `001` appears (or next number for that year/Folder ID) |
| TC-054 | FR-054 | Dispatch | Click Dispatch on a draft | File renamed to next sequential number, moved to `Outward/{Year}/{FolderID}/`, record appears in Outward Register |
| TC-121 | FR-121 | Approve deletion | Admin approves a pending deletion | Record and file permanently removed |

If a test fails: note the FR ID, and point the agent at that specific function (file + line, or just the function name) plus the error message, and ask for a fix. **Do not** re-trigger a full re-read of the requirements/design pair for a single-function bug fix unless the fix needs broader context.

**③ What to feed into AI to execute Phase 2C**
- `IODMS_requirements_v2.md`
- `IODMS_design.md` — now including the signed-off Testing Log section from this phase

---

## Phase 2C — LAN Network Design

**① What to feed in**
- `IODMS_requirements_v2.md` — specifically: air-gapped LAN, central server PC, ~30 users / max 10 concurrent, Chromium-only (`NFR-008`/`NFR-009`/`NFR-010`), no client-side install.
- `IODMS_design.md` — with the passed Testing Log section already inside it, confirming the standalone base is stable.

**② What to do in this step**
Very little new code. What changes from standalone:

- FastAPI is configured to listen on the server PC's LAN IP, not just localhost.
- PostgreSQL is configured to accept connections from other PCs on the same LAN.
- Session handling is verified with up to 10 simultaneous users (no timeout enforced, per `NFR-003`).
- A one-page deployment guide is written for accessing the app from a Chromium browser with zero install.

AI appends a new **Section E — LAN Deployment Config** to `IODMS_design.md` (connection string, `uvicorn` startup command, `pg_hba.conf` patch, deployment guide) — still the same single document, not new config files scattered around.

Your job: read the deployment section. It must match the physical setup — the server PC's fixed LAN IP, the configured IODMS root path (`NFR-140`/`EIR-003`), and Chromium 109 as the minimum supported client (`NFR-009`).

**③ What to feed into AI to execute Phase 3B**
- `IODMS_requirements_v2.md`
- `IODMS_design.md`

---

## Phase 3B & 3C — LAN Code Changes and Testing

**① What to feed in**
- `IODMS_requirements_v2.md`
- `IODMS_design.md`

**② What to do in this step**
Rerun every test from the Testing Log section — this time from a different PC on the network, pointing the browser at the server PC's LAN IP. Append results as new rows in the same Testing Log section.

Add these LAN-specific tests:
- Two users editing different records at the same time — both changes save correctly
- Two users attempting to open the same draft in MS Word at the same time — second user sees the "currently being edited by [User Name]" message (`FR-052`)
- Auditor View accessed from a second PC with no credentials — register loads read-only, watermarked, copy-paste disabled (`FR-001`–`FR-007`)

Pass all tests → Deployment done.

**③ What to feed into AI after deployment**
No further AI sessions needed for standard deployment. If a bug appears post-deployment: open the project in Claude Code/Antigravity, point it at the specific function with its FR comment plus the error message. Nothing else — never the whole design document for a single function fix.

---

## What You Actually Need to Learn (Realistic, in Order)

You do not need to write code from scratch. You need to read it and understand it enough to debug it with AI help.

**Before Phase 2B starts (next few weeks):**
- What is an API and what do GET / POST / PUT mean — one 20-minute video
- What is a database table, PRIMARY KEY, FOREIGN KEY — one hour, read examples
- Basic Python syntax — functions, variables, if/else — just enough to read it
- What `bcrypt` password hashing means, in one sentence (you'll see it in `auth.py`)

**While reviewing AI-generated code:**
- FastAPI: what does `@router.get("/something")` mean
- React: what is a component, what does `useState` do — just read, don't memorise
- PostgreSQL: how to run `SELECT * FROM inward_register` in DBeaver to check if data saved
- What "RBAC" means and why `auth.py` checks `role` before letting a request through

**That's it for now.** As each phase comes, you learn what that phase needs.

---

## Tools to Install

Per `NFR-011`, install **exact pinned versions** below — not "latest" — on whichever machine each tool belongs to. Two different machines are involved now: your **dev machine** (ordinary laptop/PC, has internet) and the **standalone PC** (will become air-gapped; per `NFR-014`, download its installers on an internet-connected machine first and carry them over by USB).

**On your dev machine (needs internet — this is where Claude Code/Antigravity run):**

| Tool | Exact version to pin | What it does |
|---|---|---|
| Claude Code | latest stable | Terminal-first agentic coding tool; reads/writes your project files directly |
| Google Antigravity | latest stable, public preview | Full agent-first IDE (VS Code fork); alternative to Claude Code, or use alongside it |
| Node.js | 20.11.1 (LTS) | Needed if you use Antigravity's built-in browser to visually test React pages |
| Git | latest stable | Version history / undo mistakes; both agentic tools work best inside a git repo |
| VS Code | latest stable | Only needed if you pick Claude Code without Antigravity (Antigravity already includes a VS Code-based editor) |

> ℹ️ Pick one tool or both. Claude Code is closer to the "minimum file count, heavily-commented, single-file-per-module" philosophy this guide pushes — it tends to be more surgical. Antigravity adds a built-in browser for visually checking the React UI and can run multiple agents at once (see the autonomy warning two sections up) — useful, but keep it on Agent-assisted/Review-driven mode for this project.

**On the standalone PC (will become the air-gapped server):**

| Tool | Exact version to pin | What it does |
|---|---|---|
| Python | 3.11.9 | Runs the FastAPI backend |
| PostgreSQL | 15.7 | The database |
| DBeaver | latest stable at time of install (`EIR-005` requires it pre-installed on the server machine; also used for the "Open in DBeaver" button, `FR-142`) | View/query the database visually |
| Node.js | 20.11.1 (LTS) | Needed to build the React frontend for deployment |
| Microsoft Word | whatever version the office already runs | Required client-side for opening/editing `.doc` files (`EIR-004`) — not installed by you, just confirm it's present |

Install in the order listed. PostgreSQL before DBeaver. None of the AI coding tools go on this machine — only the finished, built code does, carried over by USB.

> ⚠️ Reminder: per `NFR-008`/`NFR-009`, the only browser you test the frontend in is a **Chromium 109+** browser (Chrome, Edge, or Opera). Don't reach for any JS/CSS feature newer than that baseline — flag it to the agent if you're not sure.

---

## What to Do Right Now (Next Steps in Order)

1. ✅ SRS done
2. ✅ `IODMS_requirements_v2.md` ready
3. ✅ This workflow guide ready
4. → Create the project folder on your dev machine, put `IODMS_requirements_v2.md` in it, `git init`, add `CLAUDE.md`/`AGENTS.md` with the rules block from the "If You're Using Claude Code or Antigravity" section
5. → Open the folder in Claude Code or Antigravity, ask it to generate the **Design Document** (`IODMS_design.md` — Architecture + Schema + File Structure, all three sections)
6. → Review it against the FR table and terminology reference; fix anything wrong before generating code
7. → Begin **code generation**, one file at a time in the dependency order above — `CLAUDE.md`/`AGENTS.md` handles the context, you just name the file and review the diff each time
8. → Install tools on the standalone PC while the agent is generating the design document

---

## One Rule to Remember

**Never let the agent see more than `IODMS_requirements_v2.md` and `IODMS_design.md` as project context.**
Agentic tools have no memory between sessions either — they just re-read whatever `CLAUDE.md`/`AGENTS.md` points them to, which is why that file exists. If you ever find yourself asking the agent to create a third standalone reference document, stop: that content belongs as a new section *inside* `IODMS_design.md`. Keeping it to two files is what stops the agent from contradicting itself across a long project, and it's the same discipline that keeps the codebase itself down to 20 files instead of 30+.