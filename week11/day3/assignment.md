# Week 11 - Day 3 Assignment

## Title
CRM REST API With Pagination, Search, And Status Filter

## Overview
Yesterday's bot collects leads but there is nowhere to view them yet. Today you add a CRUD REST API for leads on top of SQLite, with pagination, search by name, and status filtering. This is the repeated-pattern half of the week where AI is welcome.

## Learning Objectives Assessed
- Design a SQL schema for `leads`, `conversations`, `messages`
- Write REST endpoints for GET list, GET one, PATCH status
- Add pagination via `?limit` and `?offset`
- Add search via `?q`
- Connect the bot's state machine to real DB writes

## Prerequisites
- Days 1-2 completed

## AI Usage Rules

**Ratio:** 45/55. **Habit:** New tech = manual, patterns = AI. See [../ai.md](../ai.md).

- **ALLOWED FOR:** CRUD routes, pagination, search, status filter. Repeated patterns you have built before.
- **NOT ALLOWED FOR:** The phone-to-lead mapping logic (the bridge between the bot and the DB). You designed the state machine by hand; the connection back into the DB must also be deliberate.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Install SQLite and create schema

**What to do:**
```bash
npm install better-sqlite3
```

Create `db/schema.sql`:

```sql
CREATE TABLE IF NOT EXISTS leads (
  id TEXT PRIMARY KEY,
  wa_phone TEXT NOT NULL UNIQUE,
  name TEXT,
  email TEXT,
  inquiry_type TEXT,
  status TEXT NOT NULL DEFAULT 'new',
  notes TEXT,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS conversations (
  id TEXT PRIMARY KEY,
  lead_id TEXT REFERENCES leads(id),
  state TEXT NOT NULL,
  data TEXT,
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS messages (
  id TEXT PRIMARY KEY,
  lead_id TEXT REFERENCES leads(id),
  direction TEXT NOT NULL,
  body TEXT,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

Write the schema by hand.

**Expected output:**
`db/schema.sql` committed. Load it with a small init script.

### Task 2: Hook the bot into the DB

**What to do:**
Replace the in-memory `sessions` Map with SQL-backed state. On every message:
1. Look up or create the lead by phone number.
2. Load the conversation state from the `conversations` table.
3. Advance the state machine.
4. Save the conversation back.
5. Insert the inbound message into `messages`.

This is the bridge between the bot and the CRM. Write it by hand -- it is the glue that belongs to you.

**Expected output:**
Completing a conversation via WhatsApp creates one row in `leads`, one in `conversations`, and several in `messages`.

### Task 3: CRUD endpoints

**What to do:**
Build these routes. AI-assisted is fine:

- `GET /api/leads?limit=10&offset=0&q=&status=` -- list with filters
- `GET /api/leads/:id` -- single lead with conversation history
- `PATCH /api/leads/:id` -- update status or notes
- `GET /api/stats` -- counts by status

**Expected output:**
Four endpoints. Test each with curl or Postman. Screenshots `day3-list.png` and `day3-detail.png`.

### Task 4: Pagination and search

**What to do:**
Ensure `GET /api/leads` supports:
- `?limit=20` (default 10, max 100)
- `?offset=0`
- `?q=search` (matches against name OR email)
- `?status=new|contacted|qualified|closed`

Return `{ data: [...], total: N, limit, offset }`.

**Expected output:**
Pagination works. Search works.

### Task 5: Split log

**What to do:**
In `AI_AUDIT.md`, classify each feature:
- Schema (new-tech-adjacent, manual)
- Phone-to-lead mapping (manual, bridge logic)
- CRUD routes (AI-assisted, repeated pattern)
- Pagination helper (AI-assisted, repeated pattern)

**Expected output:**
Audit updated.

## Stretch Goals (Optional - Extra Credit)

- Add a `DELETE /api/leads/:id` endpoint that soft-deletes (sets `deleted_at`).
- Add composite filter: `?status=new&q=peter`.
- Return total counts alongside the paged data for client-side pagination.

## Submission Requirements

- **What to submit:** Repo with schema, bot-to-DB bridge, REST routes, screenshots, `AI_AUDIT.md`.
- **Deadline:** End of Day 3

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Schema loaded and tables created | 15 | Three tables with correct constraints. |
| Bot writes leads to DB (bridge) | 25 | Happy-path conversation creates DB rows. Hand-written bridge. |
| CRUD endpoints working | 25 | All four routes respond correctly. |
| Pagination and search | 15 | `?limit`, `?offset`, `?q`, `?status` all working. |
| Split log current | 15 | Every feature classified with reasoning. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Letting AI design the bridge between bot and DB.** That logic is the core of the product. Write it yourself so you know it.
- **Leaking SQL details in the API response.** Never return raw DB column names that expose implementation. Map to a clean JSON shape.
- **Forgetting to escape `q`.** Use parameterised queries to avoid SQL injection: `db.prepare("SELECT * FROM leads WHERE name LIKE ?").all("%" + q + "%")`.

## Resources

- Day 3 reading: [The CRM REST API.md](./The%20CRM%20REST%20API.md)
- Week 11 AI boundaries: [../ai.md](../ai.md)
- better-sqlite3 docs: https://github.com/WiseLibs/better-sqlite3
