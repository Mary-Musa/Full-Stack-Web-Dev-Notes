# Week 23 - Day 4 Assignment

## Title
Migrate Notifications To Queues

## Overview
Day 4 is a real migration. You take the synchronous notification sends you have been doing inline (email, WhatsApp, SMS) and move them behind a queue. Request handlers return faster, failures retry automatically, and you can inspect the pipeline.

## Learning Objectives Assessed
- Identify every inline notification call in your codebase
- Replace each with a queue enqueue
- Keep the worker close to provider docs
- Measure the latency improvement

## Prerequisites
- Days 1-3 completed

## AI Usage Rules

**Ratio this week:** 35% manual / 65% AI
**Habit:** You design the work shape, AI fills in. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Refactor mechanics.
- **NOT ALLOWED FOR:** Deciding which calls are safe to queue (some are not).
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Inventory

**What to do:**
Grep for every call to your email / WhatsApp / SMS sender across the backend. List them in `notification-inventory.md` with file:line and whether they are safe to queue (hint: notifications yes, payment initiations no).

**Expected output:**
Inventory committed.

### Task 2: Refactor one at a time

**What to do:**
For each inline call you listed, replace with an enqueue:

```javascript
// Before
await sendWhatsApp(phone, "Your order is received");

// After
await notificationQueue.add("whatsapp", { phone, body: "Your order is received" });
```

Commit each refactor separately (`refactor: queue order confirmation whatsapp send`).

**Expected output:**
All inline sends replaced.

### Task 3: Unified worker

**What to do:**
One worker, multiple job types:

```javascript
new Worker("notifications", async (job) => {
  switch (job.name) {
    case "whatsapp": return sendWhatsApp(job.data.phone, job.data.body);
    case "email":    return sendEmail(job.data.to, job.data.subject, job.data.body);
    case "sms":      return sendSms(job.data.phone, job.data.body);
  }
}, { connection, limiter: { max: 20, duration: 1000 } });
```

**Expected output:**
Single worker handles all three.

### Task 4: Latency before/after

**What to do:**
Measure `/orders` handler latency before and after the refactor (`curl -w "%{time_total}\n"`). Record five samples each in `latency.md`.

**Expected output:**
`latency.md` shows improvement.

### Task 5: Worker deployment notes

**What to do:**
In `deploy-notes.md`, answer:
- Does the worker process run in the same container as the API? Why or why not?
- How do you restart a worker without losing in-flight jobs? (Graceful shutdown: stop accepting new, finish current.)
- What happens to the queue if Redis is down?

Your own words.

**Expected output:**
`deploy-notes.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Add metrics (jobs per minute, failure rate) to a simple Express `/metrics` endpoint.
- Implement graceful shutdown on SIGTERM.
- Run two worker processes and verify they share the queue.

## Submission Requirements

- **What to submit:** Repo, inventory, latency file, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 4

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Inventory | 15 | Every inline send listed. |
| Refactored | 30 | All inline sends queued. |
| Unified worker | 20 | Handles three types. |
| Latency comparison | 20 | Before/after numbers. |
| Deploy notes | 10 | Three questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Queuing a payment initiation.** The user needs the confirmation now. Keep it inline.
- **Forgetting the consumer process.** The queue fills up silently if nobody is reading.
- **Same Redis used for queue and session cache.** Separate databases or separate Redis instances.

## Resources

- Day 4 reading: [Migrating Notifications to Queues.md](./Migrating%20Notifications%20to%20Queues.md)
- Week 23 AI boundaries: [../ai.md](../ai.md)
