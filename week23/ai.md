# Week 23 - AI Boundaries

**Ratio this week: 35% Manual / 65% AI**
**Habit introduced: "Prototype alone, productionize with AI."**
**Shift from last week: Slight dip up because BullMQ + Redis is new infrastructure.**

This week you add BullMQ for async job processing: retries, priorities, rate limits, dead letter queues, and worker processes. It is the natural extension of Week 21 cron, except now jobs are queued, distributed, and observable.

The habit: for the first prototype of any queue pattern, you work alone. Get a job into the queue, get a worker to process it, see the result. Once that first loop works, AI can help you add priorities, retries, rate limits, and dashboards.

---

## What You MUST Do Manually (35%)

### Day 1 -- Why queues, BullMQ intro
- Read the BullMQ docs. Start with "Getting Started".
- Install Redis (you already have it from Week 13). Install BullMQ.
- Create your first queue by hand. Enqueue a job. Process it in a worker. Log both sides.

### Day 2 -- Retries, priorities, delayed jobs
- Configure retry with backoff. Design the retry policy on paper first.
- Priority jobs: a high-priority job should jump ahead of queued normals.
- Delayed jobs: a job that runs in 10 minutes, not now. Useful for "send reminder after 2 hours".

### Day 3 -- Rate limiting and dead letter queues
- Rate limit a queue (e.g., "no more than 20 WhatsApp messages per second").
- Dead letter queue: where failed jobs go after exhausted retries. Inspect manually.

### Day 4 -- Migrating notifications to queues
- Move your existing notification sends (WhatsApp, SMS, Telegram) from synchronous calls to queue-based.
- Benefits: retries on failure, rate-limited, isolated from request path.

---

## What You CAN Use AI For (65%)

- Queue config (retry schedules, backoff, concurrency).
- Worker process wrappers.
- Migration of existing notification code to the queue pattern.
- Dashboard for queue stats (Bull Board or custom).

Forbidden for new-tech work on Day 1 only. Days 2-4 open up.

---

## Things AI Is Bad At This Week

- **Exactly-once delivery.** BullMQ offers at-least-once. AI sometimes claims exactly-once. It is not.
- **Worker concurrency.** AI defaults to concurrency 1 or "as many as possible". Pick deliberately based on downstream limits.
- **Memory in long-running workers.** Memory leaks show up after days, not minutes. Monitor.

---

## Core Mental Models For This Week

- **A queue is a durable list of jobs.** Durable means: survives process restarts.
- **A worker is a process that pulls from the queue.** Multiple workers = parallelism.
- **A job has a lifecycle:** queued -> active -> completed OR failed -> (retry) -> dead letter.
- **At-least-once means idempotent handlers are mandatory.** Your worker may see the same job twice.

---

## This Week's AI Audit Focus

List every job type you created with: its priority, retry policy, rate limit, and who designed those settings.

---

## Assessment

- Live: enqueue 100 jobs. Watch the workers process them. Kill one worker mid-run; observe recovery.
- Facilitator asks: "what happens if a WhatsApp job fails 5 times?" You walk the path to the DLQ.

---

## One Sentence To Remember

"Prototype with the minimum, productionize with the policy."
