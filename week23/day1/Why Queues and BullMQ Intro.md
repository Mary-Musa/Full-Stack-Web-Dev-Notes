# Week 23, Day 1: Why Queues and BullMQ Intro

> **AI boundaries this week:** 35% manual / 65% AI. Habit: *Prototype alone, productionize with AI.* See [ai.md](../ai.md).

By the end of today, you have BullMQ installed, a first queue running, and you understand the three problems queues solve that direct function calls cannot.

**Prior-week concepts you will use today:**
- Redis (Week 13, Day 2)
- The outbox worker (Week 18, Day 2)
- Cron jobs (Week 21)

**Estimated time:** 3 hours

---

## The Week Ahead

| Day | Focus |
|---|---|
| Day 1 (today) | BullMQ basics: queues, jobs, workers. |
| Day 2 | Retries, concurrency, priorities, delayed jobs. |
| Day 3 | Rate limiting, dead letter queues, worker shutdown. |
| Day 4 | Migrating the chama notifications + bulk SMS to BullMQ. |
| Day 5 | Recap. |
| Weekend | Apply queues to Project 6: the multi-channel notification hub. |

---

## Three Problems Queues Solve

**1. Work that must happen but not right now.** Sending 10,000 SMSs when an order goes out should not block the HTTP response. A queue lets you say "do this later" and move on.

**2. Work that might fail and must retry.** A WhatsApp API call fails with a 429. You want it retried in 30 seconds, then 2 minutes, then 10 minutes, then escalated. A queue with retry policies handles all of that without a single if/else in your code.

**3. Work that must be rate-limited.** You cannot send more than 20 Telegram messages a second. If your code generates 1000 messages in one burst, something has to buffer them and drip them out at the right pace. A queue is that buffer.

You have been faking all three of these with cron + outbox + `setTimeout`. It works for small volumes. BullMQ does it cleanly at any scale.

---

## What BullMQ Is

BullMQ is a Node library that uses Redis as a queue backend. Producers add "jobs" to a queue; workers pull jobs off and process them. Each job has data, a status, retry metadata, and optional delays.

It is not the only option -- `node-resque`, `agenda`, and `bree` exist. BullMQ is the one with the best docs, the most active maintenance, and the cleanest API. Every Node payment or notification system at scale uses it.

### Install

```bash
npm install bullmq
```

BullMQ requires Redis 6.2+ for some features. Check your version with `redis-cli info server | grep version`. If you are on 6.0 (what `apt install redis-server` ships on Ubuntu 20.04), upgrade.

---

## Your First Queue

Create `server/queues/index.js`:

```javascript
// server/queues/index.js
const { Queue } = require("bullmq");

const connection = {
  host: process.env.REDIS_HOST || "localhost",
  port: parseInt(process.env.REDIS_PORT || "6379", 10),
};

const notificationsQueue = new Queue("notifications", { connection });
const paymentsQueue = new Queue("payments", { connection });
const reportsQueue = new Queue("reports", { connection });

module.exports = {
  notificationsQueue,
  paymentsQueue,
  reportsQueue,
};
```

Three queues, each a named channel. Queue names become Redis keys under the hood. You can have one queue with many job types or many queues -- prefer many queues when different jobs have different failure/retry/rate semantics.

Add a job to the queue:

```javascript
const { notificationsQueue } = require("./queues");

await notificationsQueue.add("sendWhatsApp", {
  to: "+254712345678",
  message: "Your order is confirmed.",
});
```

That is it. The job is now in Redis waiting to be picked up. No worker is running yet, so nothing happens. When you start a worker, it processes the job.

---

## Your First Worker

A worker is a process that reads jobs from one queue and runs them:

```javascript
// server/workers/notifications.worker.js
const { Worker } = require("bullmq");
const whatsapp = require("../services/whatsapp.service");

const connection = {
  host: process.env.REDIS_HOST || "localhost",
  port: parseInt(process.env.REDIS_PORT || "6379", 10),
};

const worker = new Worker("notifications", async (job) => {
  console.log(`Processing job ${job.id} of type ${job.name}`);

  if (job.name === "sendWhatsApp") {
    await whatsapp.sendTextMessage({ to: job.data.to, text: job.data.message });
  } else if (job.name === "sendSMS") {
    // ... SMS sender
  } else {
    throw new Error(`Unknown job name: ${job.name}`);
  }
}, { connection });

worker.on("completed", (job) => {
  console.log(`Job ${job.id} completed`);
});

worker.on("failed", (job, err) => {
  console.error(`Job ${job.id} failed:`, err.message);
});

console.log("Notifications worker running");
```

Run it as a separate process:

```bash
node server/workers/notifications.worker.js
```

It connects to Redis, polls the `notifications` queue, processes jobs, and never exits. This is a long-lived background process -- same shape as the Express server but purposefully single-purpose.

You can also run the worker in-process inside Express (just `require` the worker file). That works for a single-server setup. For multi-server you want the worker as a separate process (Week 24 territory).

---

## A Full Example: Order Confirmation Via Queue

Before BullMQ, your order flow went:

```
payment callback -> update db -> send WhatsApp -> done
```

With a queue, it becomes:

```
payment callback -> update db -> add job to queue -> done
                                          |
                          (worker picks up) |
                                          v
                                  send WhatsApp -> done
```

The HTTP handler is faster (no waiting for WhatsApp) and the send is retryable.

Replace the existing callback code:

```javascript
// Was:
await whatsapp.sendTextMessage({ to: order.customer_phone, text });

// Becomes:
await notificationsQueue.add("sendWhatsApp", {
  to: order.customer_phone,
  text,
});
```

The callback replies to Safaricom in milliseconds; the WhatsApp send happens whenever the worker gets to it (usually under a second). And if the WhatsApp API is down, BullMQ retries automatically.

---

## Seeing Jobs

BullMQ has a web dashboard called Bull Board. Install:

```bash
npm install @bull-board/express @bull-board/api
```

Wire it into the Express app:

```javascript
// server/index.js
const { createBullBoard } = require("@bull-board/api");
const { BullMQAdapter } = require("@bull-board/api/bullMQAdapter");
const { ExpressAdapter } = require("@bull-board/express");

const serverAdapter = new ExpressAdapter();
serverAdapter.setBasePath("/admin/queues");

createBullBoard({
  queues: [
    new BullMQAdapter(notificationsQueue),
    new BullMQAdapter(paymentsQueue),
    new BullMQAdapter(reportsQueue),
  ],
  serverAdapter,
});

app.use("/admin/queues", requireAdmin, serverAdapter.getRouter());
```

Visit `/admin/queues` and you see a dashboard with every queue, its pending jobs, completed jobs, failed jobs, and retry buttons. This is free observability you would otherwise have to build.

---

## Checkpoint

1. `npm install bullmq` succeeds.
2. Adding a test job to `notificationsQueue` puts it in Redis (verify with `redis-cli KEYS "bull:notifications:*"`).
3. Starting the worker processes the job and logs completion.
4. `/admin/queues` shows the three queues with any jobs you have added.
5. Adding a job with invalid data throws in the worker, and Bull Board shows the job as failed with the stack trace.
6. Retrying a failed job from Bull Board runs it again.

Commit:

```bash
git add .
git commit -m "feat: bullmq queues worker and bull board dashboard"
```

---

## What You Learned

- Queues decouple producers from consumers.
- BullMQ uses Redis; install and connect.
- Workers are long-lived processes that process jobs.
- Bull Board gives you a free web dashboard.
- The shift from "do it now" to "add to queue" is the whole mental model.

Tomorrow: retries, concurrency, priorities, and delayed jobs.
