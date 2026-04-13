# Week 22 - Day 2 Assignment

## Title
Chama Bot Core -- WhatsApp Commands And Member Enrolment

## Overview
Today you wire a WhatsApp bot on top of the Chama tables from Day 1. Members send commands like `BALANCE`, `JOIN <code>`, `CONTRIBUTE`, and the bot responds. You reuse the WhatsApp webhook infrastructure from Week 13.

## Learning Objectives Assessed
- Parse a command from a WhatsApp message
- Enroll a new member via an invite code
- Return balances and cycle info
- Keep the bot idempotent across duplicate webhooks

## Prerequisites
- Day 1 completed
- Week 13 WhatsApp webhook still functional

## AI Usage Rules

**Ratio this week:** 25% manual / 75% AI
**Habit:** You model the domain yourself. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Command parser scaffolds, response templates.
- **NOT ALLOWED FOR:** Authorization checks on who can run which command.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Invites table

**What to do:**
Add a migration:

```sql
CREATE TABLE invites (
  code TEXT PRIMARY KEY,
  chama_id UUID NOT NULL REFERENCES chamas(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  used_at TIMESTAMPTZ
);
```

Add `scripts/create-invite.js` that inserts an invite with a short random code.

**Expected output:**
Invite inserted, code printed.

### Task 2: Command router

**What to do:**
In `bot/chama-commands.js`, parse incoming text:

```javascript
function parse(text) {
  const [cmd, ...args] = text.trim().toUpperCase().split(/\s+/);
  return { cmd, args };
}

async function handle(phone, text) {
  const { cmd, args } = parse(text);
  switch (cmd) {
    case "JOIN":     return handleJoin(phone, args[0]);
    case "BALANCE":  return handleBalance(phone);
    case "CONTRIBUTE": return handleContribute(phone);
    case "HELP":     return helpText();
    default:         return "Unknown command. Send HELP.";
  }
}
```

Mount it inside your WhatsApp webhook from Week 13.

**Expected output:**
Sending `HELP` from WhatsApp returns the help text.

### Task 3: JOIN flow

**What to do:**
When a user sends `JOIN ABC123`:
- Look up the invite; if missing or used, return an error.
- If a member with this phone already exists in the chama, return "already in".
- Otherwise insert into `members`, mark the invite as used, return "welcome".

Test end-to-end from your real WhatsApp number.

**Expected output:**
New member row created via the bot.

### Task 4: BALANCE flow

**What to do:**
Return something like:

```
Mctaba Savings Group
Cycle 1 (ends 2026-04-21)
Contributed: KES 1,500 / 2,000
Fines: KES 0
```

Query the open cycle and the member's contribution for that cycle.

**Expected output:**
Balance message correct.

### Task 5: Idempotency notes

**What to do:**
Meta redelivers webhooks. If the user's `JOIN ABC123` arrives twice, you must not create two member rows.

In `idempotency-check.md`, answer:
- How does the `UNIQUE (chama_id, phone)` constraint save you here?
- What would break if you did not have it?
- Where else in the Chama bot do you rely on a UNIQUE constraint for safety?

Your own words.

**Expected output:**
`idempotency-check.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Add a `MEMBERS` command that only the treasurer can run.
- Add rate limiting per phone (max 10 commands per minute).
- Log every command into a `command_log` table.

## Submission Requirements

- **What to submit:** Repo with bot, migration, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 2

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Invites table and script | 15 | Invite created. |
| Command router | 20 | Parses and dispatches. |
| JOIN flow | 25 | New member via bot. |
| BALANCE flow | 20 | Correct numbers. |
| Idempotency notes | 15 | Three questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Trusting the display name over the phone number.** Always identify members by `wa_id`.
- **Letting anyone run treasurer commands.** Check the phone belongs to the treasurer role.
- **Not handling double-delivery.** Always rely on UNIQUE constraints, not application-level checks.

## Resources

- Day 2 reading: [Chama Bot Core.md](./Chama%20Bot%20Core.md)
- Week 22 AI boundaries: [../ai.md](../ai.md)
