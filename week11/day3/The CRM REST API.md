# Week 11, Day 3: The CRM REST API

By the end of today, your server will expose everything the sales dashboard needs: a searchable, filterable, paginated list of leads, a detail endpoint that includes the full WhatsApp conversation, a PATCH endpoint for updating status and notes, and a stats endpoint that powers the summary cards at the top of the dashboard. You will test every route with Thunder Client before writing a single line of React.

Today is the calmest day of the week. No new APIs, no webhooks, no cryptography -- just REST routes on top of the database you have been filling up for the last two days. Everything you write today is a direct application of the Week 10 Day 1 patterns: `router.get`, `router.patch`, prepared statements, validation, 404 on missing. The only genuinely new ingredients are **pagination**, **dynamic filtering**, and the **PATCH** verb with partial updates.

**Prior-week concepts you will use today:**
- Express routing, `req.params`, `req.query`, `req.body` (Week 10, Day 1)
- Prepared statements with `?` placeholders (Week 10, Day 1)
- `router.get`, `router.post`, 404 / 400 / 500 status codes (Week 10, Day 1)
- Query parameters (`?search=...&status=...`) -- first seen when consuming APIs in Week 9
- `GROUP BY` and `COUNT(*)` in SQL -- introduced conceptually in Week 10

**Estimated time:** 3-4 hours

---

## Recap: What You Built Yesterday

On Day 2 you built:

- An outgoing WhatsApp message helper (`services/whatsapp.js`) wrapping Meta's Graph API.
- A state machine bot (`services/bot.js`) that walks users through name -- email -- inquiry -- confirmation -- complete.
- HMAC-SHA256 signature verification on every incoming webhook.
- A full audit log of inbound and outbound messages in the `messages` table.

By now your `leads.db` should contain at least one complete lead from your own testing. Today we expose that data to a dashboard.

---

## The Lead Lifecycle

Before you write the routes, understand the thing you are modelling. Every lead in a CRM moves through a pipeline. Ours looks like this:

```
  New  ->  Contacted  ->  Qualified  ->  Converted
                  \
                   ->  Lost
```

- **New** -- The bot captured the details. Nobody from the sales team has seen it yet.
- **Contacted** -- A salesperson has called or WhatsApp'd the lead back.
- **Qualified** -- The lead is a real potential customer (budget, interest, timing all make sense).
- **Converted** -- The lead bought, signed, or paid. Money moved.
- **Lost** -- The lead went cold, changed their mind, or was a time-waster.

Every CRM you will ever see is a variation of this. The names change ("Won", "Closed", "Dead") but the shape is the same: a small set of statuses, one active at a time, changed by a human.

Our PATCH route today is how statuses change. It is the most important route of the day -- the sales team will hit it dozens of times an hour.

### Why We Enforce Statuses In The Route, Not The Schema

SQLite has no native `ENUM` type. On PostgreSQL you could write a `CHECK (status IN ('new', 'contacted', ...))` constraint and the database would reject invalid values directly. On SQLite we enforce the allowed set in the Express route with a simple JavaScript array. This is the portable equivalent -- same safety, slightly different layer.

In a team with multiple services all writing to the same database, pushing the check down to SQL is safer (no service can forget to validate). In a single-service project like ours, route-level validation is fine.

---

## The Leads Routes

Replace the placeholder `server/routes/leads.js` from Day 1 with the real implementation:

```javascript
// server/routes/leads.js
const express = require("express");
const db = require("../db");

const router = express.Router();

const VALID_STATUSES = [
  "new",
  "contacted",
  "qualified",
  "converted",
  "lost",
];

// GET /api/leads?search=&status=&page=&pageSize=
router.get("/", (req, res) => {
  const search = (req.query.search || "").trim();
  const status = (req.query.status || "").trim();
  const page = Math.max(1, parseInt(req.query.page) || 1);
  const pageSize = Math.min(100, parseInt(req.query.pageSize) || 25);
  const offset = (page - 1) * pageSize;

  const where = [];
  const params = [];

  if (search) {
    where.push("(name LIKE ? OR wa_phone LIKE ? OR email LIKE ?)");
    const like = `%${search}%`;
    params.push(like, like, like);
  }
  if (status && VALID_STATUSES.includes(status)) {
    where.push("status = ?");
    params.push(status);
  }

  const whereSql = where.length ? `WHERE ${where.join(" AND ")}` : "";

  const total = db
    .prepare(`SELECT COUNT(*) AS n FROM leads ${whereSql}`)
    .get(...params).n;

  const rows = db
    .prepare(
      `SELECT * FROM leads
       ${whereSql}
       ORDER BY created_at DESC
       LIMIT ? OFFSET ?`
    )
    .all(...params, pageSize, offset);

  res.json({ total, page, pageSize, leads: rows });
});

// GET /api/leads/:id -- lead detail + full conversation
router.get("/:id", (req, res) => {
  const lead = db
    .prepare("SELECT * FROM leads WHERE id = ?")
    .get(req.params.id);

  if (!lead) return res.status(404).json({ error: "Lead not found" });

  const messages = db
    .prepare(
      `SELECT id, direction, body, created_at
       FROM messages
       WHERE lead_id = ?
       ORDER BY created_at ASC`
    )
    .all(req.params.id);

  const conversation = db
    .prepare("SELECT state, last_message_at FROM conversations WHERE lead_id = ?")
    .get(req.params.id);

  res.json({ ...lead, messages, conversation });
});

// PATCH /api/leads/:id -- update status, notes, assigned_to
router.patch("/:id", (req, res) => {
  const lead = db
    .prepare("SELECT id FROM leads WHERE id = ?")
    .get(req.params.id);
  if (!lead) return res.status(404).json({ error: "Lead not found" });

  const { status, notes, assigned_to } = req.body;

  if (status && !VALID_STATUSES.includes(status)) {
    return res.status(400).json({
      error: `status must be one of: ${VALID_STATUSES.join(", ")}`,
    });
  }

  const fields = [];
  const params = [];
  if (status !== undefined) {
    fields.push("status = ?");
    params.push(status);
  }
  if (notes !== undefined) {
    fields.push("notes = ?");
    params.push(notes);
  }
  if (assigned_to !== undefined) {
    fields.push("assigned_to = ?");
    params.push(assigned_to);
  }
  if (fields.length === 0) {
    return res.status(400).json({ error: "No updatable fields provided" });
  }

  fields.push("updated_at = datetime('now')");
  params.push(req.params.id);

  db.prepare(`UPDATE leads SET ${fields.join(", ")} WHERE id = ?`).run(
    ...params
  );

  const updated = db
    .prepare("SELECT * FROM leads WHERE id = ?")
    .get(req.params.id);
  res.json(updated);
});

module.exports = router;
```

This is a lot of code at once. Let us walk through what is new.

### Building A WHERE Clause Dynamically

The GET route accepts an optional `search` and an optional `status`. Either, both, or neither can be present. If we wrote one big SQL statement with hard-coded placeholders, we would have no way to "skip" a filter.

The pattern is: collect a list of conditions and a list of parameters in parallel, then join the conditions with ` AND ` at the end.

```javascript
const where = [];
const params = [];

if (search) {
  where.push("(name LIKE ? OR wa_phone LIKE ? OR email LIKE ?)");
  params.push(like, like, like);
}
if (status && VALID_STATUSES.includes(status)) {
  where.push("status = ?");
  params.push(status);
}

const whereSql = where.length ? `WHERE ${where.join(" AND ")}` : "";
```

The `params` array is still positional -- each `?` in the combined SQL matches the value at the corresponding index in `params`. When we call `.get(...params)` later, they line up. You will reuse this pattern in every real API you write. Learn it now.

**Why `LIKE '%search%'` instead of full text search?** Because SQLite's FTS is overkill for a CRM with a few thousand leads. `LIKE` with wildcards is instant at that scale and needs no schema changes. If the CRM ever needs to search message bodies across millions of conversations, that is when you upgrade.

**Why the `VALID_STATUSES.includes(status)` guard inside the `if`?** To stop a malicious query parameter from injecting anything. If someone requests `?status=haha`, we silently ignore the filter instead of putting `haha` into the SQL. The prepared-statement `?` would also protect us from SQL injection, but ignoring invalid input is better UX than returning an empty list.

### Pagination

```javascript
const page = Math.max(1, parseInt(req.query.page) || 1);
const pageSize = Math.min(100, parseInt(req.query.pageSize) || 25);
const offset = (page - 1) * pageSize;
```

- `Math.max(1, ...)` -- never let `page` drop below 1.
- `parseInt(...) || 1` -- if the user sends no page or something like `?page=banana`, default to 1.
- `Math.min(100, ...)` -- cap page size at 100 so a malicious client cannot ask for a million rows.

Then in SQL: `LIMIT ? OFFSET ?`. Standard stuff. Returning the `total` count alongside the current page lets the frontend show "Page 2 of 17" without a second request.

For a few thousand rows this is fine. At tens of millions you would switch to keyset pagination ("give me rows created *after* this timestamp"), but that is a future-week problem.

### The Detail Route And Joining Conversation History

The `/:id` route returns the lead, all its messages in chronological order, and the current conversation state. Three queries, one JSON response:

```json
{
  "id": "...",
  "wa_phone": "254712345678",
  "name": "John Kamau",
  "email": "john@gmail.com",
  "inquiry_type": "viewing",
  "status": "new",
  "...": "...",
  "messages": [
    { "id": 1, "direction": "inbound",  "body": "hi",          "created_at": "..." },
    { "id": 2, "direction": "outbound", "body": "Hi! Welcome...", "created_at": "..." }
  ],
  "conversation": { "state": "complete", "last_message_at": "..." }
}
```

This shape is what the dashboard's slide-out panel will render tomorrow. A single network call gets everything the sales team needs to see at once. No second request for messages, no loading spinner inside a loading spinner.

### PATCH With Partial Updates

PATCH is different from PUT. PUT replaces the whole resource. PATCH updates *some* fields. When the sales team clicks "mark as Qualified" they only want to change status -- not accidentally wipe the notes field by sending an empty body. PATCH is the right verb here.

The dynamic `SET` clause works the same way the `WHERE` clause did:

```javascript
const fields = [];
const params = [];
if (status !== undefined) {
  fields.push("status = ?");
  params.push(status);
}
// ... etc
fields.push("updated_at = datetime('now')");
params.push(req.params.id);

db.prepare(`UPDATE leads SET ${fields.join(", ")} WHERE id = ?`).run(...params);
```

The `updated_at = datetime('now')` is always appended -- if any field changed, we want the timestamp to move. That is what `updated_at` is for.

Notice we check `=== undefined`, not falsy. That is deliberate. If someone sends `{ "notes": "" }`, they want to *clear* the notes, not skip the field. `"" !== undefined` so the empty string goes through. Small thing, matters.

---

## The Stats Route

Create `server/routes/stats.js`:

```javascript
// server/routes/stats.js
const express = require("express");
const db = require("../db");

const router = express.Router();

// GET /api/stats -- dashboard summary
router.get("/", (_req, res) => {
  const total = db.prepare("SELECT COUNT(*) AS n FROM leads").get().n;

  const today = db
    .prepare(
      `SELECT COUNT(*) AS n
       FROM leads
       WHERE date(created_at) = date('now')`
    )
    .get().n;

  const byStatusRows = db
    .prepare(
      `SELECT status, COUNT(*) AS n
       FROM leads
       GROUP BY status`
    )
    .all();

  const byStatus = byStatusRows.reduce((acc, r) => {
    acc[r.status] = r.n;
    return acc;
  }, {});

  res.json({
    total,
    today,
    byStatus: {
      new: byStatus.new || 0,
      contacted: byStatus.contacted || 0,
      qualified: byStatus.qualified || 0,
      converted: byStatus.converted || 0,
      lost: byStatus.lost || 0,
    },
  });
});

module.exports = router;
```

Three queries, one response. The dashboard calls this once on load and every time the polling loop ticks (tomorrow).

**Why zero-fill `byStatus`?** Because `GROUP BY` only returns rows for statuses that actually exist in the table. If nobody is in `converted`, the dashboard would get `undefined` and crash. Filling in zeros for missing statuses keeps the frontend boring -- which is what you want in a frontend.

**Why `date(created_at) = date('now')`?** SQLite's `date()` function strips the time portion off a datetime string. `date('now')` returns today's date in UTC. If you are in Nairobi (UTC+3), leads created between midnight and 3am local time will count as "today" in UTC even though it was yesterday locally. For a sandbox that is fine. For a production deployment you would want to store timestamps in UTC and convert on the way out -- a fix for another week.

---

## Wiring Up The New Routes

Open `server/index.js` and replace the Day 1 placeholders with the real route files:

```javascript
// server/index.js  (partial -- only the changed lines)
const webhookRoutes = require("./routes/webhook");
const leadRoutes = require("./routes/leads");
const statsRoutes = require("./routes/stats");

// ...

app.use("/webhook", webhookRoutes);
app.use("/api/leads", leadRoutes);
app.use("/api/stats", statsRoutes);
```

Remove the placeholder middleware from Day 1 that returned `{ error: "Coming on Day 3" }`. Nodemon will restart the server automatically.

---

## Testing The API

You should have at least one lead in the database from Day 2. If not, message your test number and complete a conversation first. Then open Thunder Client (or Postman) and walk through every route.

### Test 1: Health Check

Still works from Day 1:

```
GET http://localhost:5000/api/health
-> { "status": "ok", "timestamp": "..." }
```

### Test 2: Stats

```
GET http://localhost:5000/api/stats
```

Expected shape:

```json
{
  "total": 1,
  "today": 1,
  "byStatus": {
    "new": 1,
    "contacted": 0,
    "qualified": 0,
    "converted": 0,
    "lost": 0
  }
}
```

### Test 3: List All Leads

```
GET http://localhost:5000/api/leads
```

Expected:

```json
{
  "total": 1,
  "page": 1,
  "pageSize": 25,
  "leads": [
    {
      "id": "...",
      "wa_phone": "254712345678",
      "name": "John Kamau",
      "email": "john@gmail.com",
      "inquiry_type": "viewing",
      "status": "new",
      "notes": null,
      "assigned_to": null,
      "created_at": "...",
      "updated_at": "..."
    }
  ]
}
```

### Test 4: Filter By Status

```
GET http://localhost:5000/api/leads?status=new
```

Should return the same lead. Now try:

```
GET http://localhost:5000/api/leads?status=converted
```

Should return an empty array.

### Test 5: Search

```
GET http://localhost:5000/api/leads?search=John
```

Should find your lead. Try searching by part of the phone number:

```
GET http://localhost:5000/api/leads?search=254
```

### Test 6: Lead Detail With Conversation

Copy the lead `id` from Test 3 and:

```
GET http://localhost:5000/api/leads/THE_ID_HERE
```

The response should include a `messages` array with every inbound and outbound message from the conversation you had on Day 2, in chronological order. This is the data that will render the chat bubbles in tomorrow's detail panel.

### Test 7: Update Status

```
PATCH http://localhost:5000/api/leads/THE_ID_HERE
Body (JSON): { "status": "contacted" }
```

Should return the updated lead with `status: "contacted"` and a new `updated_at`. Verify by hitting GET again.

### Test 8: Invalid Status Rejected

```
PATCH http://localhost:5000/api/leads/THE_ID_HERE
Body (JSON): { "status": "banana" }
```

Should return 400 with the message *"status must be one of: new, contacted, qualified, converted, lost"*.

### Test 9: Empty Patch Rejected

```
PATCH http://localhost:5000/api/leads/THE_ID_HERE
Body (JSON): { }
```

Should return 400 with *"No updatable fields provided"*. Good -- we do not want someone hitting PATCH just to bump `updated_at`.

### Test 10: 404 On Missing Lead

```
GET http://localhost:5000/api/leads/not-a-real-id
```

Should return 404 with `{ "error": "Lead not found" }`.

✅ CHECKPOINT: All ten tests pass. Now message your test number to complete a *second* conversation (use "restart" to reset) with a different name and different inquiry type. Hit `GET /api/leads` again -- you should see two leads, sorted newest first. Try `GET /api/leads?search=<partial name>` to confirm search works across multiple records. Try `GET /api/stats` and confirm `total` and `today` reflect the new count.

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `TypeError: Cannot read properties of undefined (reading 'n')` on stats | `leads` table is empty and a query returned no row | `SELECT COUNT(*)` always returns one row. If you see this, you wrote `SELECT n FROM ...` by mistake -- go back to `COUNT(*) AS n`. |
| PATCH returns 404 even when the lead exists | URL typo | The URL is `/api/leads/THE_ID`, not `/api/leads?id=THE_ID`. Path params, not query params. |
| PATCH request arrives with an empty `req.body` | Content-Type header missing in Thunder Client | Make sure the body type is set to "JSON" in Thunder Client; it automatically sets `Content-Type: application/json` when you do. |
| Search returns no rows even though a match exists | `LIKE` is case-sensitive on some SQLite setups | SQLite's `LIKE` is case-insensitive for ASCII by default. If it is not matching, double-check the string you stored -- trailing whitespace from a copy-paste is a common cause. |
| `GET /api/leads?status=NEW` returns everything instead of filtering | Uppercase vs lowercase mismatch | Our `VALID_STATUSES` has lowercase values. Either normalise the query parameter with `.toLowerCase()` or require the frontend to use lowercase. |
| CORS error in browser | Not testing from a browser today | Tomorrow when you add the React client, you will hit CORS. For today, Thunder Client does not care about CORS. |

---

## What You Built Today

- A `GET /api/leads` route with optional full-text search across name/phone/email, optional status filtering, and pagination with a total count.
- A `GET /api/leads/:id` route that returns a lead together with its full WhatsApp conversation history and current state -- everything the detail panel will need in one call.
- A `PATCH /api/leads/:id` route with partial updates, dynamic SET clauses, and server-side enum validation on status.
- A `GET /api/stats` route with total, today-only count, and per-status breakdown.
- The dynamic WHERE-clause-building pattern you will reuse in every filterable API for the rest of your career.
- Ten passing Thunder Client tests covering happy paths, search, filter, pagination, invalid input, and 404s.

Good time to commit: `git add . && git commit -m "feat: CRM REST API (leads + stats)"`

**Tomorrow:** The dashboard. You will scaffold a React + Tailwind client, build stats cards, a searchable filterable table, and a slide-out detail panel with one-click status updates and a "Reply via WhatsApp" button. Everything you built today gets wired to screens a real sales team could actually use. You will finish the week with polling (so new leads appear without a refresh), keyboard shortcuts, and a handful of stretch goals you can pick from in the last hour.
