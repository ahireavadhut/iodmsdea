# IODMS – V-Model Workflow Guide
## Who this is for: First-year Aviation intern, non-CSE background, learning as you go
## Stack: React + FastAPI + PostgreSQL

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

## Phase 1 — Requirements ✅ DONE

You interviewed the stakeholders (by looking at the old software), wrote down every screen, every field, every behaviour. That became the SRS.

**Outputs you already have:**
- SRS Word document
- Requirements context markdown (with FR-001 to FR-054)
- This workflow file

---

## Phase 2A — Standalone PC Design

**What standalone means:** The server and the user are on the same computer. No network, no other PCs. This keeps things simple for the first build. You are just making sure the software works correctly before worrying about multiple users on a network.

This phase has 4 steps. Each step produces a document. AI generates each document — your job is to read it, check it makes sense, and ask questions if something doesn't match what you know about the software.

---

### Step 1 — High Level Design (HLD)

**① What to feed in**
- `IODMS_requirements_context.md` — the AI needs FR-001 to FR-054 and the 3 main workflows (create outward file, inward entry, finalise file) to know what it is designing.
- Nothing else. No code, no schema yet — none of that exists.

**② What to do in this step**
AI generates a 2-page HLD document. It describes the 3 system components and how they connect:

```
Browser (React)  →  FastAPI (Python)  →  PostgreSQL (Database)
                          ↕
                     File System
                  (the .docx files)
```

- **Browser (React):** What the user sees and clicks on.
- **FastAPI:** The middleman — receives requests from the browser, talks to the database, returns answers.
- **PostgreSQL:** Stores every record — inward, outward, address book, users, all dropdown master lists.
- **File system:** Folders on the PC where .docx files are saved and moved.

For each of the 3 main workflows, the HLD traces the data flow end to end: what the user clicks, what FastAPI does, what goes into the database, what file moves where.

Your job: Read it. If a flow description doesn't match how you remember the old software working, flag it before moving on.

**③ What to feed into AI to execute the next step (Step 2)**
- `IODMS_requirements_context.md`
- The HLD document just generated

---

### Step 2 — Database Schema

**① What to feed in**
- `IODMS_requirements_context.md` — contains the full table list and what each stores.
- The HLD document from Step 1 — shows which tables each workflow touches, so column choices are consistent with the flow.

**② What to do in this step**
AI generates a single `schema.sql` file. Each table is defined with `CREATE TABLE` statements — every column has a name, data type, and a comment explaining what it holds. AI also generates a simple text diagram showing which tables reference which (foreign key links).

Tables AI will define:

| Table | What it stores |
|---|---|
| `users` | Officer accounts, roles, passwords |
| `designations` | Dropdown options for Designation field |
| `organisations` | Dropdown options for Organisation field |
| `folder_types` | Dropdown options for Folder Type field |
| `address_groups` | The Categories used in Create File |
| `address_book` | All addresses (migrated from legacy XLSX) |
| `inward_register` | All inward records (migrated from legacy XLSX) |
| `outward_register` | All finalised outward records (migrated from legacy XLSX) |
| `draft_files` | Temporary outward drafts waiting to be finalised |
| `audit_log` | Every action any user takes, with timestamp |

Your job: Cross-check every dropdown in the app against the table list, and every form field against the column list. Use the FR table in the requirements markdown for this check.

**Tool to view the schema once created:** Use **DBeaver** (free). Connect it to PostgreSQL and browse your tables like a spreadsheet. Simpler than pgAdmin — install that, not pgAdmin.

**③ What to feed into AI to execute the next step (Step 3)**
- `IODMS_requirements_context.md`
- `schema.sql` — Step 3 needs to know every file name and which module it belongs to, and the schema defines the data layer that each module file will talk to.

---

### Step 3 — Folder and File Structure

**① What to feed in**
- `IODMS_requirements_context.md`
- `schema.sql` from Step 2 — the folder structure must have one model file per table and one router file per module, so the schema drives the file list.

**② What to do in this step**
AI generates a full folder tree — every folder and file listed, with a one-line comment on each file explaining its purpose. No code is written yet. This is just the skeleton.

Why this matters: AI generates code one file at a time. If the folder structure isn't locked first, it won't know where to import things from and files will contradict each other.

Make sure the tree has one folder/file per module (Create File, Inward Entry, Finalise File, Address Book, Admin, Auth) so every FR can be traced to a specific file.

Your job: Confirm the tree matches the module list you know from the SRS. If a screen is missing its own file, ask AI to add it.

**③ What to feed into AI to execute the next step (Step 4)**
- `IODMS_requirements_context.md`
- `schema.sql`
- The folder/file structure document — each code generation prompt references a specific file path from this tree.

---

### Step 4 — Code Generation (One Module at a Time)

**① What to feed in — per module**

For each file you ask AI to generate, feed:
- `IODMS_requirements_context.md`
- `schema.sql`
- The folder/file structure document
- The already-written files that the current file depends on (listed in the dependency order below)

Do **not** paste all previously written files at once — only the ones this specific file imports from.

**② What to do in this step**

Generate code in this dependency order (things other things need go first):

1. Database connection setup
2. All database table models (Python version of the schema)
3. Authentication (login/session) — everything else depends on this
4. Address Book + Admin master lists — dropdowns need these before other modules can use them
5. Inward Entry and Inward Register
6. Create File + document generator
7. Finalise File + file mover
8. Outward Register
9. Frontend pages — same order as backend

Use this prompt pattern for every file:

```
I am a first-year non-CSE intern building a document management system
for HAL AURDC Nashik. I am learning as I go.

Project context: [paste IODMS_requirements_context.md]
Schema: [paste schema.sql]
Folder structure: [paste folder/file structure doc]

Now generate: backend/routers/inward.py
Requirements this file covers: FR-016 to FR-033
Files already written that this depends on: database.py, models/inward.py

Rules:
- Add a comment above every function saying which FR ID it implements
- Keep comments simple enough for a non-CSE beginner to understand
- If something is complex, explain it in a comment before the code
```

Every function must have an FR comment:

```python
# FR-016: When user clicks New, generate the next Inward Number automatically
@router.post("/inward/new")
def create_inward_entry(...):
    ...
```

This is how you trace every line of code back to a requirement. When something breaks, find the FR, find the function, fix it.

**③ What to feed into AI to execute Phase 3A**
- `IODMS_requirements_context.md`
- The folder/file structure document (to know which file implements which module)
- The complete FR list (already in the requirements markdown)
- **Do not** paste all generated code — test cases are generated from requirements, not from code.

---

## Phase 3A — Standalone Testing

**① What to feed in**
- `IODMS_requirements_context.md` — the full FR list is the source of truth for what gets tested.
- The folder/file structure document — so AI knows which module to reference per test case.

**② What to do in this step**
AI generates a test case document — one row per FR. You run each test manually on the standalone PC and mark pass or fail.

| TC ID | FR ID | What you test | What you do | What should happen |
|---|---|---|---|---|
| TC-001 | FR-001 | Subject field | Type text into Subject | Text appears correctly |
| TC-016 | FR-016 | Inward No. auto-generate | Click New | INWD-2025-001 appears |
| TC-013 | FR-013 | Finalise File | Click Finalise | Register No. generated, record moves to Outward Register |

If a test fails: note the FR ID, find the function in the code with that FR comment, paste only that function + the error to AI and ask for a fix.

**③ What to feed into AI to execute Phase 2C**
- `IODMS_requirements_context.md`
- The signed-off test results document (listing all passed TCs)
- `schema.sql` — LAN design adds network config on top of the same schema; no table changes expected.

---

## Phase 2C — LAN Network Design

**① What to feed in**
- `IODMS_requirements_context.md` — specifically the deployment context: air-gapped LAN, central server PC, ~30 users, max 10 concurrent, Chromium browser on client PCs, no client-side install.
- Passed test results from Phase 3A — confirms the standalone base is stable before adding network layer.
- `schema.sql` — network config changes touch DB connection settings; AI needs the existing schema to generate correct config patches.

**② What to do in this step**
Very little new code. What changes from standalone:

- FastAPI is configured to listen on the server PC's LAN IP, not just localhost.
- PostgreSQL is configured to accept connections from other PCs on the same LAN.
- Session handling is verified to work correctly with up to 10 simultaneous users.
- A simple install/access guide is written so other PCs can open the app in their browser with no install.

AI generates: updated config files (`database.py` connection string, `uvicorn` startup command, `pg_hba.conf` patch) and a one-page deployment guide.

Your job: Read the deployment guide. It must match the actual physical setup — the server PC's fixed LAN IP, the folder where .docx files are stored, and the Chromium browser on client machines.

**③ What to feed into AI to execute Phase 3B**
- `IODMS_requirements_context.md`
- Updated config files generated in this step
- Phase 3A test case document — Phase 3B reruns the same tests from a different PC, so AI extends that document with LAN-specific additions.

---

## Phase 3B & 3C — LAN Code Changes and Testing

**① What to feed in**
- `IODMS_requirements_context.md`
- Phase 3A test case document
- Updated config files from Phase 2C

**② What to do in this step**
Rerun every test from Phase 3A — this time from a different PC on the network, pointing the browser at the server PC's LAN IP.

Add these LAN-specific tests:

- Two users editing different records at the same time — both changes save correctly.
- Session timeout works across the network — user is logged out after inactivity.
- Scanner integration test (if scanner is connected and integration is complete by this phase).

Pass all tests → Deployment done.

**③ What to feed into AI after deployment**
No further AI sessions needed for standard deployment. If a bug appears post-deployment: paste `IODMS_requirements_context.md` + the specific function with its FR comment + the error message. Nothing else.

---

## What You Actually Need to Learn (Realistic, in Order)

You do not need to write code from scratch. You need to read it and understand it enough to debug it with AI help.

**Before Phase 2B starts (next few weeks):**
- What is an API and what do GET / POST / PUT mean — watch one 20-minute YouTube video
- What is a database table, what is a PRIMARY KEY, what is a FOREIGN KEY — one hour, just read examples
- Basic Python syntax — functions, variables, if/else — not deep, just enough to read it

**While reviewing AI-generated code:**
- FastAPI: what does `@router.get("/something")` mean
- React: what is a component, what does `useState` do — just read, don't memorise
- PostgreSQL: how to run `SELECT * FROM inward_register` in DBeaver to check if data saved

**That's it for now.** As each phase comes, you learn what that phase needs. Don't try to learn everything before starting.

---

## Tools to Install (Standalone PC, in This Order)

| Tool | What it does | Where to get it |
|---|---|---|
| Python 3.11+ | Runs the FastAPI backend | python.org |
| PostgreSQL 15+ | The database | postgresql.org |
| DBeaver | View and query your database visually (simpler than pgAdmin) | dbeaver.io |
| Node.js 20+ | Needed to run React | nodejs.org |
| VS Code | Code editor | code.visualstudio.com |
| Cursor | AI-powered code editor, best tool for this project | cursor.sh |
| Git | Saves versions of your code so you can undo mistakes | git-scm.com |

Install in the order listed. PostgreSQL must be installed before DBeaver.

---

## What to Do Right Now (Next Steps in Order)

1. ✅ SRS done
2. ✅ Requirements markdown ready
3. ✅ This workflow guide ready
4. → Start new chat, paste requirements markdown, ask for **HLD document**
5. → Then paste requirements markdown + HLD, ask for **DB Schema** (`schema.sql` + table relationship diagram)
6. → Then paste requirements markdown + schema, ask for **folder and file structure** with comment headers
7. → Then begin **code generation**, one module at a time, using the prompt pattern in Step 4
8. → Install tools on the standalone PC while AI is generating design documents

---

## One Rule to Remember

**Feed AI the requirements markdown at the start of every new session.**
AI has no memory between conversations. Every time you start fresh, paste the context file first. That's the only way it knows what system it's building.
