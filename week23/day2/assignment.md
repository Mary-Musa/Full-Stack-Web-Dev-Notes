# Week 23 - Day 2 Assignment

## Title
Retries, Priorities, And Delays

## Overview
Real workers fail. Today you configure bounded retries with exponential backoff, prioritise urgent jobs over backfill jobs, and schedule jobs to run in the future.

## Learning Objectives Assessed
- Set `attempts` and `backoff` on a queue job
- Use `priority` to re-order the queue
- Use `delay` to schedule work for later
- Handle permanent vs transient failures

## Prerequisites
- Day 1 completed

## AI Usage Rules

**Ratio this week:** 35% manual / 65% AI
**Habit:** You design the work shape, AI fills in. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Options boilerplate.
- **NOT ALLOWED FOR:** Choosing the retry strategy for money-touching work.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Retry with backoff

**What to do:**
Add default job options to the queue:

```javascript
const emailQueue = new Queue("email", {
  connection,
  defaultJobOptions: {
    attempts: 5,
    backoff: { type: "exponential", delay: 1000 },
    removeOnComplete: 100,
    removeOnFail: 500,
  },
});
```

In the worker, throw on purpose for the first three calls to test.

**Expected output:**
Job retries 1s, 2s, 4s, 8s between attempts. Succeeds on the fourth.

### Task 2: Priorities

**What to do:**
Queue two jobs:

```javascript
await emailQueue.add("welcome", { to }, { priority: 10 });
await emailQueue.add("backfill", { to }, { priority: 1 }); // lower priority
```

Watch the worker pick the priority-1 job last.

**Expected output:**
Lower-priority jobs wait.

### Task 3: Delays

**What to do:**
```javascript
await emailQueue.add("nudge", { to }, { delay: 60000 }); // 60s
```

The job sits in the delayed set and fires in one minute.

**Expected output:**
Job fires exactly when expected.

### Task 4: Permanent vs transient failure

**What to do:**
Not every failure should retry. "Email bounced, bad address" is permanent -- retrying wastes work.

In the worker, throw `UnrecoverableError` from BullMQ for permanent failures:

```javascript
const { UnrecoverableError } = require("bullmq");
if (badEmail) throw new UnrecoverableError("Bad email");
```

In `retry-strategy.md`, write 5-7 sentences:
- What is a transient failure? (Timeout, 503, rate limit.)
- What is a permanent failure? (400 bad request, 4xx validation error.)
- Why should permanent failures NOT retry?
- Why do BullMQ's `UnrecoverableError` and normal errors behave differently?

Your own words.

**Expected output:**
`retry-strategy.md` committed.

### Task 5: Pattern audit

**What to do:**
List every job type in your system so far and classify each:

| Job | Attempts | Backoff | Priority | Notes |
|-----|----------|---------|----------|-------|
| welcome email | 5 | exp 1s | 10 | transient retries |
| ... | | | | |

Commit as `job-policy.md`.

**Expected output:**
`job-policy.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Add `maxStalledCount` to detect dead workers.
- Implement custom backoff by passing a function.
- Configure a job with `repeat: { cron: "0 8 * * *" }`.

## Submission Requirements

- **What to submit:** Repo, notes, policy doc, `AI_AUDIT.md`.
- **Deadline:** End of Day 2

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Retry with backoff | 25 | Exponential, bounded. |
| Priorities | 15 | Order respected. |
| Delays | 15 | Fires at right time. |
| Permanent vs transient | 20 | UnrecoverableError used. |
| Retry strategy notes | 15 | Four questions answered. |
| Job policy table | 5 | Filled. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Infinite retries.** Always cap `attempts`.
- **Retrying validation errors forever.** Use UnrecoverableError for 4xx.
- **Fixed delay instead of exponential.** Thundering herd on recovery.

## Resources

- Day 2 reading: [Retries Priorities and Delays.md](./Retries%20Priorities%20and%20Delays.md)
- Week 23 AI boundaries: [../ai.md](../ai.md)
