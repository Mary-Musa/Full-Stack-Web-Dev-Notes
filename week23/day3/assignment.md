# Week 23 - Day 3 Assignment

## Title
Rate Limiting And Dead Letter Queues

## Overview
Day 3 is about respecting the services you call. WhatsApp Cloud API allows N messages per second. Your worker must stay under that. And when a job fails permanently after all retries, it goes to a dead-letter queue for human review.

## Learning Objectives Assessed
- Configure worker-level rate limiting
- Build a dead-letter queue
- Move exhausted jobs to it
- Inspect failed jobs for debugging

## Prerequisites
- Days 1-2 completed

## AI Usage Rules

**Ratio this week:** 35% manual / 65% AI
**Habit:** You design the work shape, AI fills in. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Worker options boilerplate.
- **NOT ALLOWED FOR:** Deciding the rate limit values (you look up real provider limits).
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Rate limit the worker

**What to do:**
```javascript
new Worker("whatsapp", processor, {
  connection,
  limiter: { max: 20, duration: 1000 }, // 20 per second
});
```

Look up your WhatsApp provider's real limit and cite the source in a comment.

**Expected output:**
Worker respects the limit. No 429s from WhatsApp.

### Task 2: DLQ setup

**What to do:**
Create a second queue `whatsapp-dlq`. In the worker's `failed` handler, if a job has reached max attempts, copy the payload and error to the DLQ:

```javascript
worker.on("failed", async (job, err) => {
  if (job.attemptsMade >= job.opts.attempts) {
    await dlqQueue.add("failed-whatsapp", { payload: job.data, error: err.message, failedAt: new Date().toISOString() });
  }
});
```

**Expected output:**
Exhausted jobs land in DLQ.

### Task 3: DLQ inspector

**What to do:**
Add an admin endpoint `GET /admin/dlq` that lists failed jobs and `POST /admin/dlq/:id/retry` that moves one back to the main queue. Protect behind admin auth.

**Expected output:**
DLQ inspector works.

### Task 4: Notes

**What to do:**
In `dlq-notes.md`, write 5-7 sentences:
- Why a DLQ instead of just deleting failed jobs?
- What should you do with DLQ items weekly?
- What risk is there in "just replay everything in the DLQ"?

Your own words.

**Expected output:**
`dlq-notes.md` committed.

### Task 5: Load test

**What to do:**
Queue 1,000 WhatsApp jobs in a loop. Watch the worker. Confirm it processes at (approximately) your rate limit. Make one job fail permanently on purpose and confirm it reaches the DLQ.

**Expected output:**
Observed rate and DLQ landing recorded in `load-test-notes.md`.

## Stretch Goals (Optional - Extra Credit)

- Build a small Next.js page for DLQ visualization.
- Use BullMQ's built-in `failed` state instead of a separate queue and compare the tradeoffs.
- Page yourself when DLQ depth crosses 10.

## Submission Requirements

- **What to submit:** Repo, notes, load test, `AI_AUDIT.md`.
- **Deadline:** End of Day 3

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Rate limit configured | 20 | With cited source. |
| DLQ setup | 25 | Exhausted jobs routed. |
| DLQ inspector | 25 | List + retry endpoints. |
| DLQ notes | 15 | Three questions answered. |
| Load test observations | 10 | Rate and DLQ recorded. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Rate limit on queue instead of worker.** It belongs on the worker.
- **DLQ with no human review process.** A growing DLQ means silent failures.
- **Retrying DLQ items blindly.** Investigate root cause before replay.

## Resources

- Day 3 reading: [Rate Limiting and Dead Letter Queues.md](./Rate%20Limiting%20and%20Dead%20Letter%20Queues.md)
- Week 23 AI boundaries: [../ai.md](../ai.md)
