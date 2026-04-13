# Week 29 - Day 4 Assignment

## Title
Unified Inbox And Dashboard Polish

## Overview
Day 4 builds a unified inbox that surfaces WhatsApp and USSD events in one dashboard view, plus a payments list backed by wallet_entries.

## Learning Objectives Assessed
- Display multi-channel events in a unified view
- Query across two tables with a UNION
- Paginate mixed results
- Polish the dashboard for the demo

## Prerequisites
- Days 1-3 completed

## AI Usage Rules

**Ratio this week:** 25% manual / 75% AI
**Habit:** Integrations week. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Query shapes, table rows.
- **NOT ALLOWED FOR:** RLS verification.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Unified inbox view

**What to do:**
`GET /v1/inbox?limit=20&cursor=...` returns WhatsApp messages + USSD events for the current tenant, ordered by created_at DESC.

Use a `UNION ALL` across two tables (or a materialised view if you prefer). Each row has `{channel, from, body, createdAt, id}`.

**Expected output:**
Unified list works.

### Task 2: Inbox page

**What to do:**
Next.js Server Component `/app/(authed)/inbox/page.js` renders the list with channel badges (WA/USSD).

**Expected output:**
Page renders real data.

### Task 3: Payments page

**What to do:**
`/app/(authed)/payments/page.js` lists wallet_entries for the tenant, grouped by day, with a running total.

**Expected output:**
Renders correctly.

### Task 4: Error states

**What to do:**
Every page has a loading and an empty state. The inbox's empty state says "No messages yet. Share your USSD code or WhatsApp number."

**Expected output:**
Empty states work.

### Task 5: Polish checklist

**What to do:**
`docs/polish-checklist.md`:

```markdown
- [ ] Consistent typography and spacing
- [ ] All buttons have loading states
- [ ] All destructive actions confirm
- [ ] 404 for cross-tenant access
- [ ] Inbox paginates
- [ ] Payments grouped by day
- [ ] Mobile layout works
- [ ] Empty states on every list
```

Tick honestly.

**Expected output:**
Committed.

## Stretch Goals (Optional - Extra Credit)

- Inbox filters (channel, date).
- Export CSV of payments.
- Keyboard shortcuts.

## Submission Requirements

- **What to submit:** Repo, pages, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 4

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Unified inbox endpoint | 25 | UNION works. |
| Inbox page | 20 | Renders. |
| Payments page | 25 | Grouped + running total. |
| Error/empty states | 15 | Every page. |
| Polish checklist | 10 | Honest. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **UNION without ORDER BY.** Results become random.
- **Materialised view without a refresh plan.** Stale data.
- **Polish items ignored.** Polish is the difference between a project and a product.

## Resources

- Day 4 reading: [Unified Inbox and Dashboard.md](./Unified%20Inbox%20and%20Dashboard.md)
- Week 29 AI boundaries: [../ai.md](../ai.md)
