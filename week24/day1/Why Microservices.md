# Week 24, Day 1: Why Microservices, and When Not

By the end of today, you understand what a microservice split actually is, the five good reasons and five bad reasons people split, and a concrete plan for extracting the Notification service from your Express app as your first split.

**Prior-week concepts you will use today:**
- The Week 19 payments package (already semi-extracted)
- The Week 23 queue architecture (already decoupled)
- The layered Week 12 architecture (already modular)

**Estimated time:** 2 hours (thinking, not code).

---

## The Week Ahead

| Day | Focus |
|---|---|
| Day 1 (today) | Why and when. |
| Day 2 | Extract the notification service into its own process. |
| Day 3 | Inter-service communication: HTTP and shared queues. |
| Day 4 | Shared contracts, versioning, observability. |
| Day 5 | Recap. |
| Weekend | A small two-service system working together, deployed. |

---

## What "Microservice" Actually Means

A microservice is a separately deployable unit of your application. Instead of one big process that does everything, you have N smaller processes each doing one thing. They talk to each other over HTTP, queues, or shared databases.

You do not need 20 microservices to "do microservices". Two is a microservices architecture. A team of three can run five services comfortably.

The difference between "a module" and "a microservice" is deployment. A module is a folder; a microservice is a process you can deploy without redeploying the rest.

---

## Five Good Reasons

1. **Different scaling needs.** Your payment service handles 10 requests per second. Your image processing service handles 1 request per minute but each takes 30 seconds. Running them in the same process wastes resources.
2. **Different failure isolation.** A bug in the notification service should not crash the checkout flow. Separate processes cannot share memory corruption.
3. **Different tech stacks.** The ML service is Python; the web app is Node. You cannot merge them and you would not want to.
4. **Different teams.** Two teams working on the same process step on each other. Two teams each owning a service deploy independently.
5. **Different deployment cadences.** The notification service changes weekly; the billing service changes once a quarter. Separate deploys let each team move at their own pace.

If none of these apply to you, do not split. Microservices have a real cost.

---

## Five Bad Reasons

1. **"Monolith" sounds bad.** It does not. Monoliths are faster to develop, easier to debug, and cheaper to run. "Service-oriented" is marketing.
2. **"Netflix does it."** Netflix has 15,000 engineers. You have three. Copy their tech and you get their overhead without their budget.
3. **"It will scale."** Probably not; 99% of apps never need to scale beyond what one well-tuned Postgres can do.
4. **"Resume-driven development."** Splitting a one-person app into five services teaches the wrong lessons.
5. **"DDD said so."** Domain-Driven Design talks about "bounded contexts" which *might* become services. You can have bounded contexts inside a monolith. They are separate concepts.

The short version: you split when not splitting starts costing you more than splitting would. Not before.

---

## The Cost Of Splitting

- **Extra infrastructure.** Two processes = two deploys, two monitoring setups, two logs.
- **Network latency.** An in-process function call is microseconds; a network call is milliseconds. Multiply that across a request spanning five services and you have real latency.
- **Distributed failure modes.** "The user endpoint is up but the profile endpoint is down" -- your frontend has to handle 17 new partial-failure states.
- **Distributed transactions.** Impossible to do cleanly. If two services both need to write for the same operation, you need the saga pattern, which is hard.
- **Schema migrations across services.** Adding a field now requires coordinating two repos and two deploys.
- **Shared libraries.** Something has to hold the shared types/schemas and stay in sync.
- **Observability.** You now need distributed tracing to see where a request goes.

Every one of these is solvable. Each costs engineering time that was not previously needed.

---

## Your Codebase Is Already Halfway There

Good news: the layered architecture from Week 12 already separates concerns. The Week 19 payments package is already a separate module. The Week 23 queues already decouple producers from consumers. You are not starting from a big ball of mud.

The split we do this week is:

```
Current:
  server/ (Express)
    - routes/
    - services/
    - workers/ (BullMQ workers)
    - chama/
    - payments-package-consumer

After this week:
  api/ (Express, HTTP handling only)
    - routes/
    - controllers/
    - (calls worker service via HTTP or queue)
  notification-service/ (BullMQ worker + sender logic)
    - workers/
    - services/ (whatsapp, telegram, sms)
  chama/ (optional, could stay in api for now)
```

Two services plus the existing payments package. That is manageable.

---

## Why Start With Notifications

Notifications are the ideal first extraction because:

1. **They are already queue-based** (Week 23). The producer adds a job; the consumer processes. You can move the consumer to another process without changing the producer.
2. **They have no synchronous return value.** The caller does not care when the notification finishes; fire and forget.
3. **They are a real bottleneck.** Rate limits mean they benefit from independent scaling.
4. **They touch external APIs** that can fail. Isolating them prevents a WhatsApp outage from affecting checkout.

Other natural candidates:

- **The payment service.** Already a package; could become a separate process.
- **The USSD service.** Owns its own state machine and Redis keys.
- **The cron/scheduler.** Runs independently of HTTP.

We split notifications first this week and leave the rest for the capstone.

---

## The Split Plan

**Before:** one `server/` folder, one `npm start` that runs Express and starts the notification workers in-process.

**After:**

```
api/            # the Express HTTP server
  routes/
  controllers/
  services/     # business logic
  queues/       # producers only (Queue instances, no Workers)
  package.json

notification-service/
  workers/      # BullMQ workers
  senders/      # whatsapp, telegram, sms implementations
  queues/       # consumers (Worker instances)
  package.json
```

Two folders. Two `package.json`s. Both import the `mctaba-payments` package (if they need payments). Both talk to the same Redis and the same Postgres. They communicate via the shared BullMQ queues.

Deploy them as two processes. Day 2 actually does the split.

---

## Shared Code

Some things need to live in both services: the database schema, the outbox event types, the notification job payload shapes. Options:

1. **A shared library package** (like `mctaba-payments`). Both services depend on it.
2. **A shared git submodule.** Both services embed it. Awkward; avoid.
3. **Duplicate the code and manually sync.** Fine for small teams with good tests.

We use option 1 for contracts that change often (job shapes, database schemas) and option 3 for stable infrastructure (connection config, shared utilities). Do not overthink this on day one.

---

## Checkpoint (No Code Today)

1. You can explain in your own words what a microservice is and what it is not.
2. You can name three reasons to split and three reasons not to.
3. You can describe the cost of a split beyond "it is cooler".
4. You have a plan for tomorrow: extract notifications into a separate process sharing the same Redis and Postgres.
5. You have copied the `server/workers/` folder (mentally) to a new `notification-service/` structure and thought about what else needs to move.

Write a short `ARCHITECTURE.md` in the repo root describing the planned split. This is the document you will refer back to Day 2-4 when the code gets complicated.

Commit:

```bash
git add ARCHITECTURE.md
git commit -m "docs: plan notification-service split"
```

---

## What You Learned

- Microservices are a deployment choice, not a design style.
- There are five good reasons to split and five bad ones.
- Splitting has real costs; do the math before committing.
- Your queue-based architecture makes the split cheap.
- Notifications are the ideal first extraction.

Tomorrow we actually split.
