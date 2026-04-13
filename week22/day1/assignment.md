# Week 22 - Day 1 Assignment

## Title
Chama Project 4 -- Architecture And Data Model

## Overview
Week 22 launches Project 4: a Chama (Kenyan savings group) bot. Today you scope the project, design the database schema, and sketch the flows: member joins, contributes via M-Pesa, the cycle rotates, fines accrue, a treasurer sees the dashboard.

## Learning Objectives Assessed
- Model a rotating savings group in Postgres
- Decide cycle, member, contribution, and fine tables
- Sketch the state machine for a contribution
- Write a one-page product spec

## Prerequisites
- Weeks 1-21 completed

## AI Usage Rules

**Ratio this week:** 25% manual / 75% AI
**Habit:** You model the domain yourself. See [../ai.md](../ai.md).

- **ALLOWED FOR:** SQL boilerplate after you decided the shape.
- **NOT ALLOWED FOR:** The domain model itself -- draw it by hand first.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Product spec

**What to do:**
Create `docs/chama-spec.md`. Answer:
- Who is the user? (Chama members and one treasurer.)
- What is a "cycle"? (A round where every member contributes and one person receives the pot.)
- How does a member join? (Invite link or admin add.)
- How does a member contribute? (M-Pesa STK push.)
- What are fines? (Late contributions pay a fixed fine.)
- When does the pot pay out? (At the end of each cycle, to a rotating recipient.)

Keep it to one page. No AI on this file -- write it yourself.

**Expected output:**
`chama-spec.md` committed.

### Task 2: Draw the ERD

**What to do:**
On paper or with draw.io, sketch the entities:

```
chamas --< members
chamas --< cycles
cycles --< contributions
members --< contributions
cycles --< fines
```

Photograph or export as `docs/chama-erd.png`.

**Expected output:**
ERD committed.

### Task 3: Migrations

**What to do:**
Create migrations for `chamas`, `members`, `cycles`, `contributions`, `fines`:

```sql
CREATE TABLE chamas (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  contribution_amount_cents INTEGER NOT NULL,
  cycle_length_days INTEGER NOT NULL DEFAULT 30,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  chama_id UUID NOT NULL REFERENCES chamas(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  phone TEXT NOT NULL,
  position INTEGER NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (chama_id, phone)
);

CREATE TABLE cycles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  chama_id UUID NOT NULL REFERENCES chamas(id) ON DELETE CASCADE,
  number INTEGER NOT NULL,
  recipient_id UUID REFERENCES members(id),
  starts_at DATE NOT NULL,
  ends_at DATE NOT NULL,
  status TEXT NOT NULL DEFAULT 'open',
  UNIQUE (chama_id, number)
);

CREATE TABLE contributions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cycle_id UUID NOT NULL REFERENCES cycles(id) ON DELETE CASCADE,
  member_id UUID NOT NULL REFERENCES members(id),
  amount_cents INTEGER NOT NULL,
  mpesa_reference TEXT,
  status TEXT NOT NULL DEFAULT 'pending',
  paid_at TIMESTAMPTZ,
  UNIQUE (cycle_id, member_id)
);

CREATE TABLE fines (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cycle_id UUID NOT NULL REFERENCES cycles(id),
  member_id UUID NOT NULL REFERENCES members(id),
  amount_cents INTEGER NOT NULL,
  reason TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Run the migration.

**Expected output:**
Tables created. `\dt` shows all five.

### Task 4: Seed one chama

**What to do:**
Write `scripts/seed-chama.js` that inserts:
- One chama called "Mctaba Savings Group"
- Four members with your phone + three test numbers
- One open cycle, member 1 as recipient, ends_at = NOW() + 7 days

**Expected output:**
Seed runs cleanly.

### Task 5: Contribution state notes

**What to do:**
In `contribution-states.md`, write 5-7 sentences:
- What states does a contribution move through? (pending -> paid or pending -> failed or pending -> overdue.)
- Who transitions pending -> paid? (The M-Pesa callback.)
- What triggers pending -> overdue? (Cron, at the end of the cycle.)
- What happens to overdue contributions? (A fine row is written.)

Your own words.

**Expected output:**
`contribution-states.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Add an `invitations` table with a unique code per invite.
- Add `audit_log` for every state transition.
- Add check constraints for amounts > 0.

## Submission Requirements

- **What to submit:** Repo with spec, ERD, migrations, seed, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 1

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Product spec | 20 | One page, written by hand. |
| ERD | 15 | Covers five entities. |
| Migrations | 30 | All tables, constraints, foreign keys. |
| Seed script | 15 | Inserts chama, members, cycle. |
| Contribution state notes | 15 | Four questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Storing amounts as floats.** Always integer cents.
- **Forgetting the `UNIQUE (cycle_id, member_id)` constraint.** A member must only contribute once per cycle.
- **Letting AI invent the domain for you.** You own this model -- it reflects how the group actually works.

## Resources

- Day 1 reading: [Project 4 Architecture.md](./Project%204%20Architecture.md)
- Week 22 AI boundaries: [../ai.md](../ai.md)
