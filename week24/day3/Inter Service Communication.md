# Week 24, Day 3: Inter-Service Communication

By the end of today, you understand the four ways services talk to each other, when to use each, and which you would pick for different parts of your app. You will add a small HTTP endpoint on the notification service so the API can query delivery status synchronously.

**Prior concepts:** yesterday's split.

**Estimated time:** 2-3 hours

---

## Four Ways Services Talk

### 1. Queues (what you already have)

Service A adds a job; service B picks it up. Asynchronous, one-way, buffered.

**Pros:** decoupled, retryable, surviveable.
**Cons:** no synchronous return value; debugging is harder.

**Use for:** notifications, background processing, anything "fire and forget".

### 2. Direct HTTP

Service A calls `fetch` on service B. Service B returns JSON.

**Pros:** simple, synchronous, well-understood.
**Cons:** tight coupling, network overhead, failure cascades.

**Use for:** request-response needs, simple reads, admin actions.

### 3. Shared database

Service A writes a row; service B reads it later.

**Pros:** simplest, cheapest, fully transactional.
**Cons:** schema coupling, hard to change either side without touching the other.

**Use for:** small, stable schemas; "what you see is what you get" data.

### 4. gRPC / RPC libraries

Service A calls a typed remote procedure on service B.

**Pros:** fast, typed, observable.
**Cons:** extra infrastructure, tooling overhead, mostly overkill for small apps.

**Use for:** high-throughput, low-latency, multi-language setups. Not this Marathon.

---

## Picking The Right Shape

| Use case | Best shape |
|---|---|
| Send a WhatsApp in the background | Queue |
| Ask "what is the delivery status of message X?" | HTTP |
| Share customer data across services | Shared DB |
| Pass events from payment to notification | Queue |
| Look up a user's preferred channel | HTTP or shared DB |
| Trigger a heavy batch job | Queue |

When in doubt, default to queues. They are the most resilient.

---

## Adding A Tiny HTTP Endpoint To The Notification Service

Up to now the notification service has no web server -- it only reads the queue. Sometimes you need to ask it synchronously "is this message delivered yet?". Add a tiny Express server:

```javascript
// notification-service/httpServer.js
const express = require("express");
const { query } = require("./config/db");

const app = express();
app.use(express.json());

// Health check
app.get("/health", (req, res) => res.json({ ok: true, service: "notifications" }));

// Get delivery status for a specific message
app.get("/delivery/:attemptId", async (req, res) => {
  const { rows } = await query(
    "SELECT id, user_id, channel, status, error, created_at FROM notification_attempts WHERE id = $1",
    [req.params.attemptId]
  );
  if (!rows[0]) return res.status(404).json({ error: "Not found" });
  res.json(rows[0]);
});

// Get the user's recent attempts
app.get("/users/:userId/recent", async (req, res) => {
  const { rows } = await query(
    `SELECT id, channel, status, created_at
     FROM notification_attempts
     WHERE user_id = $1
     ORDER BY created_at DESC
     LIMIT 20`,
    [req.params.userId]
  );
  res.json(rows);
});

const port = process.env.NOTIFICATION_SERVICE_PORT || 5010;
app.listen(port, () => {
  console.log(`[notifications] HTTP server on :${port}`);
});

module.exports = app;
```

Start it from the main entry:

```javascript
// notification-service/index.js
require("./httpServer"); // starts the HTTP server
require("./workers/whatsapp.worker");
// ...
```

Now the notification service listens on port 5010 while also processing queue jobs.

### Calling it from the API

```javascript
// api/services/notifications.client.js
async function getRecentAttempts(userId) {
  const res = await fetch(`${process.env.NOTIFICATION_SERVICE_URL}/users/${userId}/recent`);
  if (!res.ok) throw new Error(`Notification service returned ${res.status}`);
  return res.json();
}

module.exports = { getRecentAttempts };
```

The API calls this when an admin loads a user's notification history page. Synchronous, one HTTP round-trip.

---

## Authentication Between Services

Internal-service HTTP should not be publicly exposed. Two common patterns:

**Internal-only network.** The notification service binds to an internal IP or a Docker network. The API can reach it but the internet cannot. Simplest, no auth needed.

**Shared secret token.** Every request carries `Authorization: Bearer <INTERNAL_TOKEN>`. The receiver rejects requests without it.

```javascript
// notification-service/httpServer.js
app.use((req, res, next) => {
  const token = req.headers.authorization?.split(" ")[1];
  if (token !== process.env.INTERNAL_SERVICE_TOKEN) {
    return res.status(401).json({ error: "Unauthorized" });
  }
  next();
});
```

Pick one. For the Marathon, shared secret is cleaner. For production with Kubernetes, internal-only networks are the default.

**Never expose service-to-service endpoints on the public internet.** A "send a WhatsApp to this number" endpoint that anyone can hit is a spam cannon.

---

## Timeouts, Circuit Breakers, Backpressure

Synchronous HTTP between services introduces new failure modes. The notification service is slow -> the API hangs -> users see timeouts -> the API queue fills up -> the API crashes.

Three mitigations, in increasing sophistication:

1. **Timeouts on every HTTP call.** Always. Not "eventually" -- from the first version.

```javascript
async function callWithTimeout(url, opts = {}, timeoutMs = 3000) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);
  try {
    const res = await fetch(url, { ...opts, signal: controller.signal });
    return res;
  } finally {
    clearTimeout(timer);
  }
}
```

2. **Fail fast.** If the notification service is known to be down (detected via health checks), skip the call entirely instead of waiting for the timeout. Cache a "service is down" flag in Redis for 30 seconds.

3. **Circuit breakers.** After N consecutive failures, open the circuit and fail immediately for a period. Libraries like `opossum` implement this.

For the Marathon, (1) is enough. (2) and (3) come up in real scale.

---

## A Minimal Retry For Transient HTTP Errors

```javascript
async function retryFetch(url, opts, { retries = 3, delayMs = 500 } = {}) {
  for (let i = 0; i < retries; i++) {
    try {
      const res = await callWithTimeout(url, opts, 3000);
      if (res.ok || res.status < 500) return res;
    } catch (err) {
      if (i === retries - 1) throw err;
    }
    await new Promise((r) => setTimeout(r, delayMs * Math.pow(2, i)));
  }
}
```

Retry only on network errors and 5xx. 4xx errors are the caller's fault and retrying will not help.

---

## Checkpoint

1. The notification service listens on port 5010 with a `/health` endpoint.
2. Calling `GET /users/user-123/recent` from the API returns a JSON list.
3. Without the `INTERNAL_SERVICE_TOKEN` header, the call returns 401.
4. Stopping the notification service makes the API call timeout cleanly in under 4 seconds.
5. You can explain when to use a queue vs an HTTP call vs the shared database.

Commit:

```bash
git add .
git commit -m "feat: notification service http endpoints with auth and timeouts"
```

---

## What You Learned

- Four shapes of inter-service communication; pick by use case.
- Adding HTTP endpoints to a worker-only service is 20 lines of Express.
- Internal service auth uses shared tokens or network isolation.
- Timeouts on every call, always.
- Queues are the default; HTTP is for synchronous needs.

Tomorrow: shared contracts, versioning, and observability across services.
