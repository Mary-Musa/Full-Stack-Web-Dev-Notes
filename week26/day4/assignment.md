# Week 26 - Day 4 Assignment

## Title
Project 6 -- Notification Hub (One Endpoint, Many Channels)

## Overview
Project 6 is a "notification hub": a single internal API that any of your services can call to send a message. It abstracts away which channel (WhatsApp, SMS, email, push) the end user prefers.

## Learning Objectives Assessed
- Design a minimal public API surface
- Route a message by user preference
- Return a consistent shape across channels
- Log every send to an audit table

## Prerequisites
- Day 3 completed

## AI Usage Rules

**Ratio this week:** 25% manual / 75% AI
**Habit:** Shipping week. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Adapter boilerplate.
- **NOT ALLOWED FOR:** The API contract (you design it).
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: API contract

**What to do:**
Design `POST /send` in `contracts/notification-hub.md`:

```markdown
POST /send
Authorization: Bearer <service token>

Body:
{
  "userId": "uuid",
  "template": "booking.confirmation",
  "data": { "name": "Alice", "slotTime": "2026-04-20T09:00:00Z" },
  "channelHint": "whatsapp"  // optional
}

Response 202:
{ "sendId": "uuid", "channel": "whatsapp", "status": "queued" }
```

The hub decides the actual channel based on user preferences + hint.

**Expected output:**
Contract committed.

### Task 2: Preferences table

**What to do:**
```sql
CREATE TABLE notification_prefs (
  user_id UUID PRIMARY KEY,
  channels JSONB NOT NULL
);
-- Example value: ["whatsapp", "email"]
```

And `notification_log`:

```sql
CREATE TABLE notification_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL,
  template TEXT NOT NULL,
  channel TEXT NOT NULL,
  status TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Expected output:**
Tables created.

### Task 3: The hub endpoint

**What to do:**
Implement `POST /send`:
- Authenticate the bearer token against a service-tokens table (or a static list for now).
- Load user prefs. Pick the first available channel (respecting `channelHint`).
- Queue the notification via Week 23's queue.
- Write a `notification_log` row with status `queued`.
- Return `202`.

**Expected output:**
Integration test passes end-to-end.

### Task 4: Template rendering

**What to do:**
Create `templates/booking.confirmation.whatsapp.txt`:

```
Hi {{name}}, your booking for {{slotTime}} is confirmed.
```

Use a tiny mustache-style replacer (or install `mustache`). Render in the worker before sending.

**Expected output:**
Real messages rendered per template and channel.

### Task 5: Wire booking -> hub

**What to do:**
From your Project 5 booking flow, instead of calling the queue directly, POST to the hub. The hub decides the channel.

**Expected output:**
End-to-end booking confirmation via hub.

## Stretch Goals (Optional - Extra Credit)

- Add retry status transitions (queued -> sent -> failed) in the log.
- Support channel-level templates for the same template key.
- Add rate limiting per user.

## Submission Requirements

- **What to submit:** Repo, contract, tables, endpoint, templates, `AI_AUDIT.md`.
- **Deadline:** End of Day 4

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Contract | 15 | Written, followed. |
| Tables | 15 | Prefs + log. |
| Hub endpoint | 30 | Auth, routing, queueing, logging. |
| Template rendering | 20 | Works per channel. |
| Booking -> hub wired | 15 | End-to-end. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **No auth on the hub.** Anyone can spam your users.
- **Template interpolation via string concat.** XSS risk in email. Use a real templater.
- **Log fills forever.** Partition or prune.

## Resources

- Day 4 reading: [Project 6 Notification Hub.md](./Project%206%20Notification%20Hub.md)
- Week 26 AI boundaries: [../ai.md](../ai.md)
