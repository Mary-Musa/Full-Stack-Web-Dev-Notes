# Week 26 - Day 3 Assignment

## Title
Project 5 -- Booking System (Scope And Build)

## Overview
Day 3 starts Project 5: a small booking system. Stylists (or trainers, or doctors -- pick one) have calendars. Customers book slots. Reminders go out. You scope it, build the data model, and ship a first-cut UI.

## Learning Objectives Assessed
- Scope a small project cleanly
- Model time slots in Postgres without timezone drama
- Build a booking UI with a calendar view
- Send confirmation and reminder via the notification queue

## Prerequisites
- Weeks 1-25 completed

## AI Usage Rules

**Ratio this week:** 25% manual / 75% AI
**Habit:** Shipping week. See [../ai.md](../ai.md).

- **ALLOWED FOR:** UI scaffolding, SQL for basic CRUD.
- **NOT ALLOWED FOR:** Timezone decisions (store UTC, render local).
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: One-page spec

**What to do:**
`docs/booking-spec.md`:
- Who is the provider? (One stylist to start.)
- What is a slot? (30-minute block on a specific day.)
- How does a customer book? (Pick a slot, enter name + phone.)
- What triggers a reminder? (1 hour before the slot.)

Keep it one page.

**Expected output:**
Spec committed.

### Task 2: Data model

**What to do:**
```sql
CREATE TABLE providers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL
);
CREATE TABLE slots (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  provider_id UUID NOT NULL REFERENCES providers(id),
  starts_at TIMESTAMPTZ NOT NULL,
  duration_minutes INTEGER NOT NULL DEFAULT 30,
  status TEXT NOT NULL DEFAULT 'open',
  UNIQUE (provider_id, starts_at)
);
CREATE TABLE bookings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slot_id UUID NOT NULL UNIQUE REFERENCES slots(id),
  customer_name TEXT NOT NULL,
  customer_phone TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Note `UNIQUE (slot_id)` on bookings -- a slot can only be booked once.

**Expected output:**
Migration runs.

### Task 3: Seed slots

**What to do:**
Write `scripts/seed-slots.js` that creates one provider and slots for the next 7 days, every 30 minutes between 09:00 and 17:00 Africa/Nairobi.

Store in UTC. Convert at render.

**Expected output:**
~112 slots seeded for the week.

### Task 4: Book a slot

**What to do:**
In your Next.js app, build `/book` that lists open slots and a form. On submit, insert the booking. The `UNIQUE (slot_id)` protects against double-booking.

When successful, queue a confirmation notification via the queue.

**Expected output:**
Slots shown, booking works, double-booking rejected.

### Task 5: Timezone notes

**What to do:**
In `booking-tz.md`, write 5-7 sentences:
- Why store `starts_at` as TIMESTAMPTZ?
- Why render slots in the user's browser timezone?
- What would go wrong if you stored `"09:00"` as text? (DST, server moves country, etc.)

Your own words.

**Expected output:**
`booking-tz.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Calendar grid UI (month/week view).
- Cancellation flow with a reason.
- Slot buffers (no back-to-back bookings).

## Submission Requirements

- **What to submit:** Repo, spec, migrations, UI, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 3

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Spec | 10 | One page. |
| Data model | 25 | Constraints enforced. |
| Seed script | 15 | ~112 slots. |
| Book flow | 30 | Double-booking rejected. |
| Timezone notes | 15 | Three questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Storing local time without zone.** Impossible to render correctly across users.
- **Double-booking race.** Rely on the UNIQUE index, not application checks.
- **No confirmation.** The user should know their booking succeeded.

## Resources

- Day 3 reading: [Project 5 Booking System.md](./Project%205%20Booking%20System.md)
- Week 26 AI boundaries: [../ai.md](../ai.md)
