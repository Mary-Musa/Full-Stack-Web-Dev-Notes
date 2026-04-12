# Week 19, Day 1: Extracting the Payments Service

By the end of today, the payments code scattered across your Express server is a self-contained, installable Node package with a clean API, its own README, its own tests, and zero knowledge of the shop. Another developer could drop it into an unrelated project and take payments in an hour.

This week's deliverable is the "reusable Payments Service module" the syllabus promises. You are not building new functionality -- you are turning existing functionality into something portable.

**Prior-week concepts you will use today:**
- The unified payments interface (Week 17, Day 3)
- Webhook security (Week 18, Day 1)
- Idempotency and outbox (Week 18, Day 2)
- Refunds and reconciliation (Week 18, Days 3-4)

**Estimated time:** 3 hours

---

## The Week Ahead

| Day | Focus |
|---|---|
| Day 1 (today) | Extract the payments code into a standalone package. |
| Day 2 | Define the public API and write the README. |
| Day 3 | Tests -- unit tests for pure functions, integration tests for providers. |
| Day 4 | Publish to a private registry, or at least a GitHub package. Install in the shop as a dependency. |
| Day 5 | Recap and peer review. |
| Weekend | Polish and a small demo project that uses the package from scratch. |

---

## Why Extract At All

You could leave the payments code in the Express server. It works. Why go through the pain of extraction?

Three reasons:

1. **Reusability.** A second shop -- or the capstone in Week 27 -- can use the same module without copy-pasting.
2. **Testability.** A package with a narrow API is easier to unit-test than code mixed with HTTP handling and business logic.
3. **Credibility.** "I built a Node package that handles Kenyan payments" is a stronger portfolio item than "I wrote some payment routes in an Express app".

The transformation makes you think about what the payments service *is*, independent of the shop -- which forces you to clarify boundaries you may have been fuzzy on.

---

## The Target Shape

```
mctaba-payments/
  package.json
  README.md
  index.js              # main entry, re-exports the public API
  src/
    providers/
      mpesa.js
      airtel.js
      stripe.js
    refund.js
    reconciliation.js
    outbox.js
    webhookVerifier.js
  test/
    mpesa.test.js
    airtel.test.js
    stripe.test.js
    refund.test.js
```

One folder, one `package.json`, everything it needs. The host app (the shop) imports from it like any other package:

```javascript
const payments = require("mctaba-payments");
```

---

## Step 1: New Folder, New Package

Outside your shop folder:

```bash
mkdir mctaba-payments
cd mctaba-payments
npm init -y
```

Edit `package.json`:

```json
{
  "name": "mctaba-payments",
  "version": "0.1.0",
  "description": "A minimal payments service for M-Pesa, Airtel, and Stripe.",
  "main": "index.js",
  "license": "MIT",
  "scripts": {
    "test": "node --test test/"
  },
  "dependencies": {
    "axios": "^1.6.0",
    "stripe": "^14.0.0"
  },
  "peerDependencies": {
    "pg": "^8.0.0"
  }
}
```

Three things worth naming.

**`peerDependencies` for `pg`.** The host app already has a Postgres pool; we do not want to create a second one or force a specific version. Instead, the host passes its pool into the module at init time. `peerDependencies` tells npm "use the pg version your host already has". Same pattern React uses -- React itself is a peer dep of every React library.

**`scripts.test` uses `node:test`**, the built-in runner, same as Week 13's USSD tests. No Jest, no Vitest, no config.

**No `type: module`**. We are using CommonJS (`require`/`module.exports`). Next.js can import CommonJS fine; ESM would introduce friction in places you do not want it.

---

## Step 2: Move The Files

Copy from `server/`:

- `server/services/payments/` -> `mctaba-payments/src/providers/` (rename folder structure to match)
- `server/services/refunds.service.js` -> `mctaba-payments/src/refund.js`
- `server/services/reconciliation.service.js` -> `mctaba-payments/src/reconciliation.js`
- `server/middleware/webhookVerifier.js` -> `mctaba-payments/src/webhookVerifier.js`
- Relevant parts of `server/workers/outbox.js` -> `mctaba-payments/src/outbox.js`

You will have to touch every import path. The files currently import `../../config/db`, `../../config/env`, etc. Those paths no longer exist. You need to change them so the module does not rely on the host's file structure.

---

## Step 3: The Init Function

The package can not just auto-load a Postgres pool or read env vars at module load time -- those belong to the host. Instead, expose an `init` function:

```javascript
// index.js
let _pool = null;
let _config = null;

function init({ pool, config }) {
  if (!pool) throw new Error("pool is required");
  if (!config) throw new Error("config is required");

  // Validate config
  const required = ["mpesa", "airtel", "stripe"];
  for (const p of required) {
    if (!config[p]) console.warn(`mctaba-payments: no config for ${p}`);
  }

  _pool = pool;
  _config = config;

  // Re-export the public API with the pool/config bound
  return {
    initiate: require("./src/initiate")(_pool, _config),
    handleCallback: require("./src/handleCallback")(_pool, _config),
    refund: require("./src/refund")(_pool, _config),
    reconcile: require("./src/reconciliation")(_pool, _config),
    verifyWebhook: require("./src/webhookVerifier")(_config),
  };
}

module.exports = { init };
```

Each internal file becomes a **function that takes `pool` and `config` and returns the real function.** That is the standard closure-based dependency injection pattern in Node. The host code calls `init` once at boot and gets back an object of ready-to-use functions.

Example of a provider adapted:

```javascript
// src/providers/mpesa.js
module.exports = function createMpesaProvider(pool, config) {
  const mpesaConfig = config.mpesa;

  async function getDarajaToken() {
    const auth = Buffer.from(`${mpesaConfig.consumerKey}:${mpesaConfig.consumerSecret}`).toString("base64");
    // ... rest as before, but reading from mpesaConfig instead of process.env
  }

  async function initiate({ orderId, phone, amountCents }) {
    const token = await getDarajaToken();
    // ... use pool for SQL and mpesaConfig for secrets
  }

  async function handleCallback(body) {
    // ...
  }

  async function refund({ orderId, amountCents, reason }) {
    // ...
  }

  return { initiate, handleCallback, refund };
};
```

The pattern is simple: one big factory function, everything it needs is in scope, no `process.env` or global state inside. Testable because you can pass a mock pool and a mock config.

---

## Step 4: Update The Host

Back in the shop's Express server, install the local package:

```bash
cd server
npm install ../mctaba-payments
```

npm adds a `file:../mctaba-payments` entry to `package.json`. In Week 19 Day 4 we replace this with a real registry install.

Now in `server/index.js`:

```javascript
const payments = require("mctaba-payments").init({
  pool: require("./config/db").pool,
  config: {
    mpesa: {
      consumerKey: process.env.MPESA_CONSUMER_KEY,
      consumerSecret: process.env.MPESA_CONSUMER_SECRET,
      shortcode: process.env.MPESA_SHORTCODE,
      passkey: process.env.MPESA_PASSKEY,
      callbackUrl: `${process.env.PUBLIC_URL}/api/payments/callback/mpesa`,
    },
    airtel: {
      clientId: process.env.AIRTEL_CLIENT_ID,
      clientSecret: process.env.AIRTEL_CLIENT_SECRET,
      country: "KE",
      currency: "KES",
    },
    stripe: {
      secretKey: process.env.STRIPE_SECRET_KEY,
      webhookSecret: process.env.STRIPE_WEBHOOK_SECRET,
    },
  },
});

// Make payments available to routes
app.locals.payments = payments;
```

And the routes call `req.app.locals.payments.initiate(...)`.

---

## Step 5: The Config Schema

The host must pass a valid config, and the module must refuse to start with a bad one. Add validation:

```javascript
// src/validateConfig.js
function validateConfig(config) {
  const errors = [];

  if (!config.mpesa?.consumerKey) errors.push("mpesa.consumerKey is required");
  if (!config.mpesa?.consumerSecret) errors.push("mpesa.consumerSecret is required");
  if (!config.mpesa?.shortcode) errors.push("mpesa.shortcode is required");
  // etc

  if (!config.stripe?.secretKey) errors.push("stripe.secretKey is required");
  if (!config.stripe?.webhookSecret) errors.push("stripe.webhookSecret is required");

  if (errors.length > 0) {
    throw new Error(`Invalid mctaba-payments config:\n - ${errors.join("\n - ")}`);
  }
}

module.exports = validateConfig;
```

Call it at the top of `init`. Now a misconfigured host fails loudly and early instead of silently, at 3am when the first payment fails.

---

## Step 6: Prove It Still Works

Run a full Week 16 flow: buy a product, pay with M-Pesa sandbox, verify the order flips to `paid`, verify the WhatsApp message arrives. Every code path goes through the new package.

Then try a Stripe payment. Then an Airtel payment. All three should work unchanged from the customer's perspective.

This is the deliverable check: the shop does not care where the payments code lives. Moving it into a package is a zero-regression refactor.

---

## Checkpoint

1. `mctaba-payments/` exists as its own folder with `package.json`, `index.js`, `src/`, `test/`.
2. The shop server imports `mctaba-payments` and calls `init()` at boot.
3. No `process.env` reads inside `mctaba-payments/src/**`. Grep to prove.
4. No `require("../../config/db")` inside `mctaba-payments/src/**`. Grep to prove.
5. Paying for an order on the shop goes through the package; the order flips to `paid`.
6. `npm test` in `mctaba-payments/` runs (even if it has no real tests yet).
7. Removing the package folder breaks the shop (confirming the dependency is real and not shadowed by old local code).

Commit inside the shop:

```bash
git add .
git commit -m "refactor: use mctaba-payments as a dependency instead of inline code"
```

Commit inside the package:

```bash
cd ../mctaba-payments
git init
git add .
git commit -m "init: first version of mctaba-payments"
```

---

## What You Learned

- Packages take `pool` and `config` as init-time dependencies, not module-load-time.
- `peerDependencies` lets a package share a dependency with the host without duplicating.
- Factory functions return closures over the injected dependencies; every function is testable with a mock.
- Config validation at init time catches misconfigurations before traffic arrives.
- A zero-regression refactor means the customer sees nothing, the code sees everything.

Tomorrow we write a proper README and document the public API for other developers who do not have you on Slack.
