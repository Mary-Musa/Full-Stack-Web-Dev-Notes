# Week 24, Day 2: Extracting the Notification Service

By the end of today, the notification workers run as a separate Node process in a separate folder with its own `package.json`, talking to the same Redis and Postgres as the API. The shop takes orders; the notification service sends messages. Killing either does not affect the other.

**Prior concepts:** yesterday's plan.

**Estimated time:** 3-4 hours

---

## The New Folder Structure

```
your-project/
  api/                       # was: server/
    package.json
    index.js                 # Express
    routes/
    controllers/
    services/
      chama/
      orders/
    queues/
      index.js               # producers only
  notification-service/      # NEW
    package.json
    index.js                 # worker entrypoint
    workers/
      whatsapp.worker.js
      telegram.worker.js
      sms.worker.js
    senders/
      whatsapp.js
      telegram.js
      sms.js
    queues/
      index.js               # consumers
  shop/                      # Next.js shop (unchanged)
  mctaba-payments/           # library (unchanged)
```

Rename `server/` to `api/`. This makes the split intent clear from the directory name.

---

## Step 1: Move Files

From `api/workers/`, move these files to `notification-service/workers/`:

- `whatsapp.worker.js`
- `telegram.worker.js`
- `sms.worker.js`

From `api/services/`, move the "sender" implementations (not the callers):

- `whatsapp.service.js` -> `notification-service/senders/whatsapp.js`
- `telegram.service.js` (the sender half) -> `notification-service/senders/telegram.js`
- SMS sender similarly.

Stuff that stays in the API:

- Controllers that enqueue jobs.
- Business services that decide what to send (still in the API because they know about orders, chamas, etc).

Stuff that moves to the notification service:

- Code that takes a "send this message to that recipient" and actually sends it.

---

## Step 2: New `package.json`s

```json
// notification-service/package.json
{
  "name": "notification-service",
  "version": "0.1.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "bullmq": "^5.0.0",
    "pg": "^8.0.0",
    "axios": "^1.6.0",
    "dotenv": "^16.0.0",
    "redis": "^4.0.0"
  }
}
```

Install in the new folder:

```bash
cd notification-service
npm install
```

---

## Step 3: The Entry Point

```javascript
// notification-service/index.js
require("dotenv").config();

const whatsappWorker = require("./workers/whatsapp.worker");
const telegramWorker = require("./workers/telegram.worker");
const smsWorker = require("./workers/sms.worker");

console.log("Notification service started");

async function shutdown() {
  console.log("Shutting down notification workers...");
  await Promise.all([
    whatsappWorker.close(),
    telegramWorker.close(),
    smsWorker.close(),
  ]);
  console.log("Done.");
  process.exit(0);
}

process.on("SIGTERM", shutdown);
process.on("SIGINT", shutdown);
```

Run it:

```bash
node index.js
```

And in another terminal, run the API:

```bash
cd ../api
npm run dev
```

You now have two processes. The API enqueues; the notification service consumes. Both talk to the same Redis.

---

## Step 4: Shared Config

Both services need Redis and Postgres credentials. Two approaches:

**Option A: separate `.env` files per service.** Simple, standard. Duplicate some keys.

**Option B: a shared `.env` at the repo root.** Both services `dotenv.config({ path: "../.env" })`. One place to change secrets.

Pick one. For the Marathon, option A is simpler and more realistic (production deployments usually have per-service env).

---

## Step 5: Shared Database Schema

The notification service reads from the same Postgres as the API. It queries `outbox`, `orders`, maybe `users` or `chama_members`. It does not *own* any tables -- the API owns the schema.

This is "shared database, separate services" pattern. It is the opposite of pure microservices purity (which says each service owns its own database). For the Marathon it is the right trade: splitting the database would be hours of migration work for a handful of fields the notification service happens to need.

The downside: a schema change in `orders` might break the notification service. Mitigation: the API and notification service live in one repo, so schema changes and notification-service updates ship together.

Week 27's capstone introduces service boundaries with their own data; for now, shared is fine.

---

## Step 6: Kill The Workers In The API

The `api/workers/` folder should no longer contain worker code -- those files moved. Delete the old files and any `require("./workers/...")` lines in `api/index.js`.

Verify:

```bash
cd api
grep -r "new Worker" .
```

Should return nothing. Workers only live in `notification-service/`.

---

## Step 7: Two-Process Run

Install `concurrently` at the repo root to run both with one command:

```bash
cd ..
npm init -y
npm install --save-dev concurrently
```

In the root `package.json`:

```json
{
  "scripts": {
    "dev": "concurrently \"npm:dev:api\" \"npm:dev:notifications\"",
    "dev:api": "cd api && npm run dev",
    "dev:notifications": "cd notification-service && npm run dev"
  }
}
```

`npm run dev` at the root starts both. Ctrl+C stops both. This is the dev workflow; production runs them on different servers or in different Docker containers.

---

## Step 8: Verification

1. Place an order on the shop. API server logs "Order created" and "Enqueued notification job".
2. Notification service logs "Processing WhatsApp job".
3. The customer receives the WhatsApp.
4. Stop the notification service (`Ctrl+C` one of the two terminals).
5. Place another order. API still works. Bull Board shows the job piled up.
6. Restart the notification service. The job drains and the customer gets their message a minute late.

This is the payoff. An API deploy no longer interrupts in-flight notifications; a notification service crash no longer breaks checkout.

---

## Step 9: Bull Board

Decide which process hosts Bull Board. Both can read the queues. Usually the API hosts it because the admin dashboard is already on the API side.

Keep the Bull Board mount in `api/index.js`. The notification service does not need its own web server.

---

## Step 10: Logging

Each process now has its own stdout. In dev with `concurrently` they are interleaved and prefixed. In production you will have two log streams. Tag your logs so they are distinguishable:

```javascript
// notification-service/index.js
const log = (msg, ...args) => console.log("[notifications]", msg, ...args);
```

Use the prefix everywhere in that service. Same for the API. When you tail two log streams side by side, the prefix tells you which is which.

For a real production system, use a structured logger (`pino`, `winston`) and ship logs to a central store. For the Marathon, prefixed console.log is enough.

---

## Checkpoint

1. The `notification-service/` folder exists with its own `package.json` and `index.js`.
2. The API's `server/workers/` folder is empty (or deleted).
3. `concurrently` starts both processes with one command.
4. Stopping the notification service leaves the API running; Bull Board shows jobs piling up.
5. A real order goes through the full flow and the WhatsApp arrives.
6. Both processes log with different prefixes.
7. `grep -r "new Worker" api/` returns nothing.

Commit:

```bash
git add .
git commit -m "refactor: extract notification-service as a separate process"
```

---

## What You Learned

- Splitting is file moves and two `package.json`s.
- Shared Redis and Postgres are fine for a two-service system.
- Bull Board stays on one service (the API) even though both read queues.
- `concurrently` is a one-line multi-process dev setup.
- The test of a successful split: stopping one process does not affect the other.

Tomorrow: how services talk to each other when one needs a synchronous answer from the other.
