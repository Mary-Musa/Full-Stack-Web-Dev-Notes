# Week 22 - Day 3 Assignment

## Title
Contributions Via M-Pesa STK Push

## Overview
Today you wire M-Pesa STK push into the Chama bot. When a member sends `CONTRIBUTE`, the bot pushes a prompt to their phone. When the callback arrives, you mark the contribution as paid.

## Learning Objectives Assessed
- Trigger an STK push from a bot command
- Write the contribution row as `pending` before the push
- Update it to `paid` when the callback confirms
- Keep the money audit trail clean

## Prerequisites
- Day 2 completed
- Week 10 M-Pesa integration still functional

## AI Usage Rules

**Ratio this week:** 25% manual / 75% AI
**Habit:** You model the domain yourself. See [../ai.md](../ai.md).

- **ALLOWED FOR:** STK push request building.
- **NOT ALLOWED FOR:** Status transitions on a contribution -- you decide those by hand.
- **AUDIT REQUIRED:** Yes. This is money.

## Tasks

### Task 1: Reuse the M-Pesa client

**What to do:**
Import your Week 10 M-Pesa client into the Chama app. Confirm you can still obtain an access token and trigger a sandbox STK push.

**Expected output:**
Sandbox STK push fires on your phone.

### Task 2: CONTRIBUTE flow

**What to do:**
When a member sends `CONTRIBUTE`:
1. Find their chama and the open cycle.
2. Insert a `contributions` row with status `pending`. Rely on the unique constraint `(cycle_id, member_id)`.
3. Call STK push with the chama amount.
4. Reply: "Check your phone for the M-Pesa prompt."

If the unique constraint fires, reply "You already have a pending contribution this cycle."

**Expected output:**
`pending` row created and STK push fires.

### Task 3: Callback updates the row

**What to do:**
Extend your Week 10 M-Pesa callback handler. When the callback matches a pending contribution (by `mpesa_reference` or checkout request ID), update the row to `paid` and set `paid_at = NOW()`.

Then send the member a WhatsApp confirmation: "Contribution received. Thank you."

**Expected output:**
End-to-end: CONTRIBUTE -> STK -> PIN -> row becomes paid -> WhatsApp confirmation.

### Task 4: Audit the flow

**What to do:**
In `AI_AUDIT.md`, add a "Money Audit" section listing every file you touched in the Chama money flow:
- The contribution insert
- The STK push call
- The callback handler
- The WhatsApp reply

For each, note: was this your code, AI-generated, or AI-assisted? Re-read each line yourself.

**Expected output:**
Money audit committed.

### Task 5: Race condition notes

**What to do:**
In `race-notes.md`, write 5-7 sentences:
- What if the callback arrives before your STK push `.then()` resolves?
- What if two CONTRIBUTE commands arrive at the same time from the same member?
- How does the UNIQUE constraint help?
- What is an idempotency key and where might one live here?

Your own words.

**Expected output:**
`race-notes.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Add a 5-minute timeout that marks stale pending rows as `expired`.
- Include the contribution ID in the STK push `AccountReference` so the callback can find it directly.
- Store the raw M-Pesa callback payload for every contribution.

## Submission Requirements

- **What to submit:** Repo, notes, `AI_AUDIT.md` with Money Audit.
- **Deadline:** End of Day 3

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| STK push reused | 15 | Sandbox works. |
| CONTRIBUTE flow | 25 | Pending row + push. |
| Callback updates row | 25 | End-to-end works. |
| Money audit | 20 | Every file listed and re-read. |
| Race condition notes | 10 | Four questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Marking the row paid before the callback confirms.** Never trust the push response. Trust the callback.
- **No reference linking pending row to callback.** Put the contribution id in `AccountReference`.
- **Letting AI write the callback without review.** This is money. Read every line.

## Resources

- Day 3 reading: [Contributions via M-Pesa.md](./Contributions%20via%20M-Pesa.md)
- Week 22 AI boundaries: [../ai.md](../ai.md)
