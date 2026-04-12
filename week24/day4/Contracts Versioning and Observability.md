# Week 24, Day 4: Contracts, Versioning, and Observability

By the end of today, you have a shared `@mctaba/contracts` package that defines every job shape and every HTTP endpoint shape, both services use it, and you have distributed tracing that shows a single request flowing from the shop to the API to the notification service.

**Prior concepts:** the extracted notification service (Day 2), inter-service HTTP (Day 3).

**Estimated time:** 3 hours

---

## The Contract Problem

Today the API adds a job like `{ to, text }` and the notification service expects `{ to, text }`. It works because one person wrote both. When two people work on them, or when you change the shape, things break silently: the API sends `{ phone, message }`, the notification service looks for `to`, and messages stop firing.

The fix: write the contracts down, in code, and have both sides import the same definitions. If you change the shape in one place, the other breaks at build time or test time, not in production.

---

## A `@mctaba/contracts` Package

Create a new folder `contracts/`:

```javascript
// contracts/package.json
{
  "name": "@mctaba/contracts",
  "version": "0.1.0",
  "main": "index.js"
}
```

```javascript
// contracts/index.js
module.exports = {
  jobs: require("./jobs"),
  errors: require("./errors"),
  apis: require("./apis"),
};
```

### Jobs

```javascript
// contracts/jobs.js

// Every job added to the notifications queue MUST match one of these shapes.
// Adding a new shape is a minor version bump.
// Changing an existing shape is a major version bump.

const WhatsAppTextJob = {
  jobName: "whatsappText",
  payload: {
    to: "string",
    text: "string",
    orderId: "string?",
  },
};

const WhatsAppTemplateJob = {
  jobName: "whatsappTemplate",
  payload: {
    to: "string",
    templateName: "string",
    components: "array",
    orderId: "string?",
  },
};

const TelegramTextJob = {
  jobName: "telegramText",
  payload: {
    chatId: "number",
    text: "string",
    parseMode: "string?",
  },
};

function validateJob(jobName, payload) {
  const jobs = [WhatsAppTextJob, WhatsAppTemplateJob, TelegramTextJob];
  const job = jobs.find((j) => j.jobName === jobName);
  if (!job) throw new Error(`Unknown job: ${jobName}`);

  for (const [key, type] of Object.entries(job.payload)) {
    const optional = type.endsWith("?");
    const baseType = optional ? type.slice(0, -1) : type;
    if (payload[key] === undefined) {
      if (!optional) throw new Error(`Missing required field: ${key}`);
      continue;
    }
    if (baseType === "string" && typeof payload[key] !== "string") {
      throw new Error(`Field ${key} must be a string`);
    }
    if (baseType === "number" && typeof payload[key] !== "number") {
      throw new Error(`Field ${key} must be a number`);
    }
    if (baseType === "array" && !Array.isArray(payload[key])) {
      throw new Error(`Field ${key} must be an array`);
    }
  }
  return true;
}

module.exports = {
  WhatsAppTextJob,
  WhatsAppTemplateJob,
  TelegramTextJob,
  validateJob,
};
```

This is a tiny schema system in plain JS. In TypeScript you would use `zod` or `io-ts`; in plain JS the above is a starting point. The goal is not perfection -- it is catching "I sent `message` but you expected `text`" bugs before they reach production.

---

## Both Services Install The Contracts

```bash
# In api/:
npm install file:../contracts

# In notification-service/:
npm install file:../contracts
```

In the API:

```javascript
const { jobs: jobsContracts } = require("@mctaba/contracts");

await notificationsQueue.add("whatsappText", { to, text }, options);

// But validate before adding:
jobsContracts.validateJob("whatsappText", { to, text });
```

In the notification service worker:

```javascript
const { jobs: jobsContracts } = require("@mctaba/contracts");

const worker = new Worker("whatsapp", async (job) => {
  jobsContracts.validateJob(job.name, job.data);
  // ... process
});
```

Now both sides check the shape. A mismatch shows up as a clear error, not a silent failure.

---

## Versioning

The contracts package has a version. When you change a shape, bump it according to semver:

- **Add an optional field** -> patch bump.
- **Add a required field** -> major bump (existing callers will break).
- **Remove a field** -> major bump.
- **Rename** -> major bump.

Both services pin the version they depend on:

```json
{
  "dependencies": {
    "@mctaba/contracts": "^0.1.0"
  }
}
```

When you release `0.2.0` with a new job type, the caller can upgrade first. The worker upgrades next. The gap is safe because adding a new job type does not break existing jobs.

In one-repo setups with both services sharing the same `package.json`, this matters less -- they always deploy together. In multi-repo setups with different teams, contract versioning is the lifeline.

---

## Observability: Request IDs

When a request flows through three services, seeing one log line is useless. You want to see the whole path for that specific request. The trick: assign a "request id" at the entry and propagate it everywhere.

```javascript
// api/middleware/requestId.js
const crypto = require("crypto");

module.exports = function requestIdMiddleware(req, res, next) {
  req.id = req.headers["x-request-id"] || crypto.randomUUID();
  res.setHeader("X-Request-Id", req.id);
  next();
};
```

Every log line in the API includes `req.id`:

```javascript
console.log(`[api][${req.id}] Processing order ${orderId}`);
```

When the API enqueues a job, include the request id in the payload:

```javascript
await notificationsQueue.add("whatsappText", {
  to, text,
  requestId: req.id,
});
```

The worker reads the request id and includes it in its logs:

```javascript
const worker = new Worker("notifications", async (job) => {
  console.log(`[notifications][${job.data.requestId}] Processing ${job.name}`);
  // ...
});
```

Now grep your logs by request id:

```bash
tail -f logs/*.log | grep "abc-123-xyz"
```

You see the whole path: API received request abc-123, enqueued job, notification service picked it up, called the WhatsApp API, replied success. Debugging goes from hours to minutes.

### OpenTelemetry (for serious observability)

For production systems, use OpenTelemetry (OTel). It is the standard for distributed tracing. It automatically captures request flows across services and exports them to tools like Jaeger, Tempo, or Honeycomb. Setting it up takes a day but gives you a flame graph of every request.

For the Marathon, request ids in logs are enough. OTel is a stretch goal for anyone who finishes the capstone early.

---

## Health Endpoints

Every service needs a health endpoint:

```javascript
// notification-service/httpServer.js
app.get("/health", async (req, res) => {
  const checks = {};
  try {
    const redis = await getClient();
    await redis.ping();
    checks.redis = "ok";
  } catch { checks.redis = "fail"; }

  try {
    await query("SELECT 1");
    checks.postgres = "ok";
  } catch { checks.postgres = "fail"; }

  const allOk = Object.values(checks).every((v) => v === "ok");
  res.status(allOk ? 200 : 503).json({ service: "notifications", checks });
});
```

Load balancers, monitors, and uptime checkers hit this endpoint. If any dependency fails, return 503 and the load balancer takes the service out of rotation.

---

## Checkpoint

1. `@mctaba/contracts` exists with at least three job shapes and the `validateJob` helper.
2. Both the API and the notification service depend on it.
3. Adding a job with an invalid shape throws before reaching Redis.
4. Every API request has a request id in the `X-Request-Id` response header.
5. Grepping logs by request id shows the full trace across both services.
6. Both services have `/health` endpoints that check their dependencies.

Commit:

```bash
git add .
git commit -m "feat: shared contracts package request ids and health checks"
```

---

## What You Learned

- Contracts are the interface between services; write them down, version them.
- A tiny validator in plain JS catches 90% of shape bugs.
- Request ids make distributed logs grep-able.
- OpenTelemetry is the serious path; request ids are the practical path.
- Health endpoints are required, not optional.

Tomorrow: the recap and the weekend runs two services in Docker.
