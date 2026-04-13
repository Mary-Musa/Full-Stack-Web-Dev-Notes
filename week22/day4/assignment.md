# Week 22 - Day 4 Assignment

## Title
Cycles, Fines, And Reminders

## Overview
Day 4 is the automation layer for the Chama. A cron closes a finished cycle, opens the next one, calculates fines for anyone who did not contribute, and pushes reminder WhatsApp messages the day before the cycle ends.

## Learning Objectives Assessed
- Close a cycle on schedule and open the next
- Calculate fines for missing contributions
- Send reminder messages to members
- Rotate the pot recipient fairly

## Prerequisites
- Days 1-3 completed
- Week 21 cron concepts fresh

## AI Usage Rules

**Ratio this week:** 25% manual / 75% AI
**Habit:** You model the domain yourself. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Cron expression and template text.
- **NOT ALLOWED FOR:** The fine calculation or the pot rotation logic.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Reminder cron

**What to do:**
Create `jobs/cycle-reminder.js`. Every day at 08:00 EAT, find open cycles where `ends_at = CURRENT_DATE + 1` and send a WhatsApp reminder to members who have not yet contributed.

```javascript
cron.schedule("0 8 * * *", async () => {
  const { rows } = await pool.query(`
    SELECT m.id, m.phone, c.id as cycle_id
    FROM cycles c
    JOIN members m ON m.chama_id = c.chama_id
    WHERE c.status = 'open' AND c.ends_at = CURRENT_DATE + INTERVAL '1 day'
      AND NOT EXISTS (
        SELECT 1 FROM contributions cn
        WHERE cn.cycle_id = c.id AND cn.member_id = m.id AND cn.status = 'paid'
      )
  `);
  for (const row of rows) await sendWhatsApp(row.phone, "Contribution due tomorrow. Send CONTRIBUTE to pay.");
}, { timezone: "Africa/Nairobi" });
```

**Expected output:**
Reminder fires for the right members only.

### Task 2: Close-cycle cron

**What to do:**
At 23:59 EAT every day, close any cycle whose `ends_at < CURRENT_DATE`:
- Mark the cycle `closed`.
- For each member without a paid contribution, insert a `fines` row (amount = chama.contribution_amount_cents / 10).
- Open a new cycle with `number + 1`, rotating the recipient to the next position.

Wrap in a transaction and a Postgres advisory lock.

**Expected output:**
Cycle closes, fines written, new cycle opens.

### Task 3: Pot rotation

**What to do:**
Write a tiny helper:

```javascript
function nextRecipient(members, previousPosition) {
  const sorted = [...members].sort((a, b) => a.position - b.position);
  const idx = sorted.findIndex(m => m.position === previousPosition);
  return sorted[(idx + 1) % sorted.length];
}
```

Unit test it with a group of five members.

**Expected output:**
Test passes for rotation across full cycles.

### Task 4: STATUS command

**What to do:**
Add a `STATUS` WhatsApp command that returns:

```
Cycle 3 (ends 2026-05-03)
Recipient: Alice
Paid: 3 / 5
You: paid
```

Query joins across `cycles`, `members`, `contributions`.

**Expected output:**
STATUS returns the right strings.

### Task 5: Pre-weekend checklist

**What to do:**
Create `CHECKLIST.md`:

```markdown
## Week 22 Pre-Weekend Checklist
- [ ] Chama tables in place
- [ ] Invites + JOIN flow
- [ ] BALANCE command
- [ ] CONTRIBUTE triggers STK push
- [ ] Callback updates contribution to paid
- [ ] Reminder cron
- [ ] Close-cycle cron with fines
- [ ] Pot rotation tested
- [ ] STATUS command
- [ ] AI_AUDIT.md money section current
```

Tick honestly.

**Expected output:**
Checklist committed.

## Stretch Goals (Optional - Extra Credit)

- Add configurable fine amount per chama.
- Allow members to pay fines separately.
- Send a celebratory message to the recipient at the end of each cycle.

## Submission Requirements

- **What to submit:** Repo, crons, tests, checklist, `AI_AUDIT.md`.
- **Deadline:** End of Day 4

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Reminder cron | 20 | Targets unpaid members only. |
| Close-cycle cron | 30 | Fines inserted, new cycle opened, transaction safe. |
| Pot rotation tested | 20 | Unit test passes. |
| STATUS command | 15 | Correct strings. |
| Checklist | 10 | Honest. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Closing a cycle without a transaction.** If fines partial-insert you end up in inconsistent state.
- **Rotation by array index.** Positions drift when a member leaves. Rotate by `position` field.
- **Silent reminder failures.** Log every WhatsApp send result.

## Resources

- Day 4 reading: [Cycles Fines and Reminders.md](./Cycles%20Fines%20and%20Reminders.md)
- Week 22 AI boundaries: [../ai.md](../ai.md)
