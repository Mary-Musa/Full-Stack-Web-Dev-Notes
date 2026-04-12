# Week 11 Weekend Project: Boda Dispatch -- A WhatsApp Delivery Request CRM

This weekend you build a second WhatsApp-powered system from scratch, solving a different problem with the same patterns. No hand-holding tutorial this time. You have the Week 11 notes, you have the CRM you built Monday through Thursday, and you have the self-check from Day 5. The project description below tells you *what* to build and *what it must do*. Every design and implementation decision is yours.

This is how the rest of your career will actually work. You will read a product spec, recognise the shape ("this is a webhook + state machine + REST + dashboard problem"), and reach for the patterns you already know. The skill being tested this weekend is not memorisation -- it is whether you can see a new problem and map it onto the patterns from this week.

**Estimated time:** 6-8 hours across Saturday and Sunday.

---

## The Problem

A boda-boda delivery operator in Nairobi runs his entire business out of WhatsApp. Restaurants in Kilimani, shops in Westlands, and individuals in Lavington all send him delivery requests as WhatsApp messages. Between lunch rush, dinner rush, and Saturday errands, he drops deliveries constantly because he cannot keep track of which pickup is where, who the customer is, and which of his three riders is free.

He asks you to build him a "dispatch board" -- a WhatsApp number that accepts delivery requests via a short conversation, and a web dashboard where he can see every active request, assign a rider, and mark deliveries as picked up, delivered, or cancelled.

You are going to build it.

---

## What The Bot Must Do

When a customer sends any message to the WhatsApp number, the bot runs this conversation:

1. Greet them. Ask where the **pickup location** is (e.g. "Java Kilimani").
2. Ask where the **drop-off location** is.
3. Ask for the **recipient's name and phone number** at the drop-off point.
4. Ask for a **brief description of the package** (e.g. "Food order, medium bag").
5. Send an interactive list asking for **urgency**: `Standard` (under 2 hours), `Express` (under 45 minutes), or `Scheduled` (time-agreed later).
6. Show a confirmation block with all five details, ask the user to reply `yes` to confirm or `restart` to start over.
7. On confirmation, save the request as a delivery and reply: *"Request received. A rider will be assigned shortly. Your reference is BD-XXXXX."* The reference should be a short unique code the customer can quote if they call in to follow up.

Requirements for the bot:

- It must use the same state-machine-in-the-database pattern from Day 2.
- It must verify Meta's HMAC signature on every POST -- same mechanism as Day 2.
- It must handle voice notes, images, and other non-text messages gracefully (re-ask, do not crash).
- The customer's WhatsApp phone number is how you identify them. The same phone should be able to submit multiple delivery requests over time -- each new `restart` or conversation after `complete` creates a *new* delivery row, not overwriting the previous one.
- All messages in both directions must be logged for later audit.

---

## What The Database Must Model

At minimum, three tables. You pick the exact columns, but they must cover:

- **customers** -- people who have messaged the service. One row per unique WhatsApp phone. Name is optional (captured from their WhatsApp profile if available).
- **deliveries** -- one row per confirmed delivery request. Must include pickup location, drop-off location, recipient name, recipient phone, package description, urgency, status (see below), a unique reference code (the `BD-XXXXX` the customer sees), `created_at`, `updated_at`, optional `assigned_rider`.
- **messages** -- audit log of every inbound and outbound WhatsApp message, linked to the customer.

You will also need a way to track the *conversation state* for in-progress requests so the bot knows what the next message means. Whether you put that in a `conversations` table, a column on `customers`, or somewhere else is your call -- justify it in the README.

### The Delivery Lifecycle

Every delivery moves through these statuses:

```
  pending  ->  assigned  ->  picked_up  ->  delivered
      \            \
       \            `->  cancelled
        `->  cancelled
```

- **pending** -- the customer confirmed the bot conversation. No rider yet.
- **assigned** -- the operator picked a rider on the dashboard.
- **picked_up** -- the rider has the package.
- **delivered** -- package arrived.
- **cancelled** -- customer or operator cancelled.

Enforce the set server-side in your PATCH route.

---

## What The REST API Must Expose

At minimum:

- `GET /api/deliveries` with pagination, search across pickup/drop-off/recipient name/phone, and status filtering. Returns `{ total, page, pageSize, deliveries }`.
- `GET /api/deliveries/:id` returns a single delivery with its linked customer details and the full WhatsApp conversation that produced it.
- `PATCH /api/deliveries/:id` accepts partial updates for `status` and `assigned_rider`. Must validate status against the enum. Must bump `updated_at`.
- `GET /api/stats` returns counts for: total deliveries, deliveries today, a per-status breakdown, and the number of deliveries currently in `pending` (the "needs action" number the operator cares most about).

Add anything extra you think the operator would use.

---

## What The Dashboard Must Show

A single-page React + Tailwind dashboard. It must have:

- A header with the app name and the operator's branding.
- A row of stats cards driven by `/api/stats`. At least: Total, Pending, Today, Delivered Today.
- A **"Needs attention" section at the top** that shows only pending-and-unassigned deliveries. This is the operator's morning inbox.
- A searchable, filterable table of all deliveries below that.
- Colour-coded status badges. Your choice of colours, but they must be consistent across the table and the detail panel.
- A slide-out detail panel when you click a row, showing:
    - Pickup, drop-off, recipient details, package description, urgency, reference code.
    - The full WhatsApp conversation (inbound vs outbound visual distinction).
    - Status buttons for one-click transitions through the lifecycle.
    - An input to assign a rider (just a free-text name for now -- user accounts come in Week 12).
    - A "Message customer via WhatsApp" button that uses `wa.me/<phone>` to open WhatsApp directly.
- Ten-second polling so new requests appear without a page refresh.

---

## Non-Negotiables

These are the things I will check first when I review your submission. Missing any of them means the core patterns from the week did not land.

1. **HMAC signature verification on the webhook.** If I send a POST to your `/webhook` with a fake body and no valid signature, it must return 401. Test this yourself.
2. **State persisted in the database, not in memory.** Restart your server in the middle of a conversation. The bot must pick up exactly where it left off when the next message arrives.
3. **Enum validation in the PATCH route.** If I send `{ "status": "teleported" }`, I must get a 400 with a message listing the valid statuses.
4. **The raw-body hook on `express.json()`.** Your `index.js` must set up `req.rawBody` so HMAC verification can work. If you skipped this and somehow "passed" signature verification, you are not actually checking signatures.
5. **Zero-fill on `/api/stats`.** An empty database must still return `{ total: 0, today: 0, byStatus: { pending: 0, assigned: 0, ... } }`. No `undefined`.
6. **Server-side pagination.** Loading the table must not fetch every row in the database. Use `LIMIT`/`OFFSET` or equivalent.
7. **Polling with a cleanup function.** Your interval must clear itself on unmount. I will check by mounting and unmounting the dashboard repeatedly and watching for leaked timers in React DevTools.
8. **A `.env.example` file in the repo, no real `.env`.** Your `META_*` secrets must not be in git history. Run `git log -- .env` -- it must return nothing.

---

## Stretch Goals (Pick What You Enjoy)

Do any of these that sound interesting. None are required. A small well-built project with one well-done stretch goal is better than a big sloppy project with five broken ones.

- **Rider ETA broadcast.** When a delivery moves to `assigned`, send a WhatsApp message back to the customer: *"Your delivery BD-XXXXX has been assigned to <rider>. ETA 30 minutes."*
- **Cancellation reason.** When the operator cancels a delivery, prompt for a reason and store it. Show it in the detail panel.
- **"Today only" toggle.** A button above the table that filters to deliveries created in the last 24 hours. One click, no typing.
- **CSV export** of all deliveries for a given date range.
- **Rider leaderboard.** A small panel showing deliveries completed per rider today.
- **Light/dark theme** using Tailwind's `dark:` variant.
- **Keyboard shortcuts.** `Escape` closes the panel (you already built this on Day 4), `/` focuses the search bar, `n` filters to pending.

---

## Deliverables

Submit a single git repository containing:

1. A `server/` folder with everything the backend needs.
2. A `client/` folder with the React dashboard.
3. A `README.md` at the root with:
    - A screenshot or GIF of the dashboard with real data in it.
    - Setup instructions -- exact commands to install and run both halves.
    - A "Design decisions" section (under 300 words) explaining: where you put the conversation state, how you generate the `BD-XXXXX` reference codes, and one thing you chose to do differently from the Week 11 CRM tutorial and *why*.
    - A "What I skipped and why" section if you left anything out. Honesty beats sweeping things under the rug.
4. A `.env.example` listing every environment variable your code reads.
5. A `.gitignore` that includes `node_modules/`, `.env`, and your SQLite database file.

Push the repo to GitHub under a name like `boda-dispatch`. Send me the link by Monday morning.

---

## How I Will Grade This

| Area | What I Check | Weight |
|---|---|---|
| Bot works end-to-end | I message your number from my phone and complete the full conversation, including a restart in the middle. | 25% |
| State machine is clean | I read `services/bot.js`. States are explicit, transitions are obvious, no global variables holding user state. | 15% |
| Webhook security | HMAC verification actually works. Fake POSTs get 401. Raw body hook is in place. | 15% |
| REST API is correct | All four endpoints exist, pagination works, PATCH validates the enum, 404 vs 400 vs 500 are used correctly. | 15% |
| Dashboard is useful | I can do the operator's three main jobs (see pending, assign a rider, update status) without reading any instructions. | 15% |
| Code quality | Route files are thin, services are reusable, no dead code, no committed secrets. | 10% |
| README | I can clone and run your project in under five minutes using only what is in the README. | 5% |

---

## A Hint For When You Get Stuck

When you hit a wall, do not start from scratch. Open the corresponding file from your Week 11 CRM and ask yourself: *what is different about this problem, and what is the same?* Ninety percent of the time, the answer is "everything is the same except the names of the tables and the shape of the conversation." The hard thinking is spotting the patterns. Once you see them, the code writes itself.

Good luck. Ship it on Sunday night.
