# Week 23 - Day 1 Assignment

## Title
Why Queues -- BullMQ And Redis

## Overview
Week 23 introduces background job queues. Today you install BullMQ, run Redis, enqueue a "send email" job from an HTTP handler, and watch a worker process it. You learn the difference between "do it now inside the request" and "put it on a queue and return fast."

## Learning Objectives Assessed
- Run Redis locally
- Install and use BullMQ
- Enqueue a job from an Express handler
- Process the job in a separate worker process

## Prerequisites
- Weeks 1-22 completed

## AI Usage Rules

**Ratio this week:** 35% manual / 65% AI
**Habit:** You design the work shape, AI fills in. See [../ai.md](../ai.md).

- **ALLOWED FOR:** BullMQ boilerplate.
- **NOT ALLOWED FOR:** Deciding what is queueable vs what stays synchronous.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Redis via Docker

**What to do:**
```bash
docker run -d --name redis -p 6379:6379 redis:7
```

Verify with `redis-cli ping` (returns PONG).

**Expected output:**
Redis running on 6379.

### Task 2: Install BullMQ

**What to do:**
```bash
npm install bullmq ioredis
```

Create `queues/email.js`:

```javascript
const { Queue } = require("bullmq");
const connection = { host: "localhost", port: 6379 };
const emailQueue = new Queue("email", { connection });
module.exports = { emailQueue, connection };
```

**Expected output:**
File committed.

### Task 3: Enqueue from a handler

**What to do:**
In an Express handler:

```javascript
app.post("/welcome", async (req, res) => {
  await emailQueue.add("welcome", { to: req.body.email });
  res.json({ queued: true });
});
```

Test with `curl`. The response returns in milliseconds.

**Expected output:**
200 response. Job visible in Redis (`redis-cli KEYS 'bull:email:*'`).

### Task 4: Worker

**What to do:**
Create `workers/email-worker.js`:

```javascript
const { Worker } = require("bullmq");
const { connection } = require("../queues/email");

new Worker("email", async (job) => {
  console.log("[worker] sending welcome email to", job.data.to);
  await new Promise(r => setTimeout(r, 500)); // pretend SMTP
  console.log("[worker] done");
}, { connection });

console.log("Email worker online");
```

Run it in a separate terminal: `node workers/email-worker.js`. Queue a job and watch it process.

**Expected output:**
Worker logs show the job processed.

### Task 5: Queue vs inline notes

**What to do:**
In `queue-vs-inline.md`, write 5-7 sentences:
- Why return 200 immediately and queue the email instead of sending inline?
- What breaks for the user if the SMTP server is slow?
- What breaks for the user if your queue worker is down? (Nothing visible -- the job sits in Redis until the worker restarts.)
- When should a job stay inline? (When the user needs confirmation right now -- payment initiation, login.)

Your own words.

**Expected output:**
`queue-vs-inline.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Add Bull Board (`@bull-board/express`) to visualise queue state.
- Queue multiple job types under one queue.
- Use `ioredis` with a password in a .env file.

## Submission Requirements

- **What to submit:** Repo with queue, worker, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 1

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Redis running | 10 | PONG. |
| BullMQ installed | 10 | In package.json. |
| Queue module | 20 | Re-usable. |
| Enqueue from handler | 20 | Fast response. |
| Worker processes jobs | 25 | Logs confirm. |
| Queue vs inline notes | 10 | Four questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Queue and worker in the same process.** They CAN share a process, but the point is to split them.
- **Forgetting the connection param.** BullMQ requires an explicit Redis connection.
- **Assuming jobs are persistent by default.** Redis without persistence loses jobs on restart. Configure AOF/RDB.

## Resources

- Day 1 reading: [Why Queues and BullMQ Intro.md](./Why%20Queues%20and%20BullMQ%20Intro.md)
- Week 23 AI boundaries: [../ai.md](../ai.md)
- BullMQ docs: https://docs.bullmq.io/
