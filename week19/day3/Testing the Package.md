# Week 19, Day 3: Testing the Package

By the end of today, `mctaba-payments` has a real test suite. Unit tests cover the pure logic (validation, signature verification, state transitions). Integration tests exercise the providers against their sandbox APIs or fakes. The `npm test` command runs in under 30 seconds and gives a clear pass/fail.

**Prior concepts:** yesterday's README; the package structure from Day 1.

**Estimated time:** 3 hours

---

## Two Kinds Of Tests

**Unit tests** run in a fraction of a second each. They test pure functions: given these inputs, return these outputs. No network, no database, no side effects.

**Integration tests** touch real systems or carefully-crafted fakes. Slower, more fragile, but they catch issues unit tests cannot see -- like "Stripe's API returns a different shape than we expected".

Aim for lots of unit tests and a handful of integration tests. 80/20 split by count.

---

## The Unit Tests

Start with the functions that take plain inputs and return plain outputs -- validators, formatters, the state-transition rules.

```javascript
// test/formatPhone.test.js
const { test } = require("node:test");
const assert = require("node:assert/strict");
const formatPhone = require("../src/formatPhone");

test("formatPhone handles +254 prefix", () => {
  assert.equal(formatPhone("+254712345678"), "254712345678");
});

test("formatPhone handles 0 prefix", () => {
  assert.equal(formatPhone("0712345678"), "254712345678");
});

test("formatPhone handles 254 prefix", () => {
  assert.equal(formatPhone("254712345678"), "254712345678");
});

test("formatPhone rejects garbage", () => {
  assert.throws(() => formatPhone("not a phone"));
});
```

If `formatPhone` does not yet exist as a standalone function, extract it. The test forces you to make it pure.

```javascript
// test/validateInitiateArgs.test.js
const { test } = require("node:test");
const assert = require("node:assert/strict");
const validate = require("../src/validateInitiateArgs");

test("rejects negative amounts", () => {
  const result = validate({ orderId: "x", phone: "+254712345678", amountCents: -1 });
  assert.ok(result.error);
});

test("rejects missing orderId", () => {
  const result = validate({ phone: "+254712345678", amountCents: 100 });
  assert.ok(result.error);
});

test("accepts valid args", () => {
  const result = validate({ orderId: "x", phone: "+254712345678", amountCents: 100 });
  assert.ok(!result.error);
});
```

Every validator, every formatter, every pure utility gets one test per branch. 20 minutes of test writing, 20 years of confidence.

---

## Fake Providers

For the callback handlers, you want to test the full flow without hitting Daraja. Inject a fake provider:

```javascript
// test/fakes/fakeMpesa.js
module.exports = function createFakeMpesa({ overrideStatus } = {}) {
  return {
    async initiate({ orderId, phone, amountCents }) {
      return {
        reference: `FAKE_CO_${Date.now()}`,
        nextStep: { type: "wait" },
      };
    },
    async handleCallback(body) {
      // Accept any body, do nothing
    },
    async queryStatus(reference) {
      return { status: overrideStatus || "success" };
    },
  };
};
```

The key insight: your package's public API takes a `method` string, but internally uses a registry. Make the registry injectable so tests can override providers:

```javascript
// src/createPayments.js
module.exports = function createPayments({ pool, config, providers }) {
  const defaultProviders = {
    mpesa: require("./providers/mpesa")(pool, config),
    airtel: require("./providers/airtel")(pool, config),
    stripe: require("./providers/stripe")(pool, config),
  };
  const activeProviders = { ...defaultProviders, ...providers }; // overrides win

  async function initiate(method, args) {
    return activeProviders[method].initiate(args);
  }
  // ...
  return { initiate, /* ... */ };
};
```

Now in tests:

```javascript
const createPayments = require("../src/createPayments");
const fakeMpesa = require("./fakes/fakeMpesa");
const fakePool = require("./fakes/fakePool");

const payments = createPayments({
  pool: fakePool,
  config: {},
  providers: { mpesa: fakeMpesa() },
});

test("initiate mpesa returns a reference", async () => {
  const result = await payments.initiate("mpesa", {
    orderId: "o1",
    phone: "+254712345678",
    amountCents: 100,
  });
  assert.ok(result.reference.startsWith("FAKE_CO_"));
});
```

No network, no real Daraja, no real database. The test runs in 5 milliseconds.

---

## The Fake Pool

A tiny in-memory Postgres replacement for tests:

```javascript
// test/fakes/fakePool.js
module.exports = {
  _rows: {
    orders: new Map(),
    outbox: [],
    webhook_events: new Set(),
  },

  async query(text, params = []) {
    // Extremely minimal SQL pretend. Enough for unit tests.
    if (text.includes("INSERT INTO outbox")) {
      this._rows.outbox.push({ id: this._rows.outbox.length + 1, payload: params[0] });
      return { rows: [{ id: this._rows.outbox.length }] };
    }
    if (text.includes("SELECT") && text.includes("orders") && text.includes("WHERE id")) {
      const order = this._rows.orders.get(params[0]);
      return { rows: order ? [order] : [] };
    }
    if (text.includes("UPDATE orders")) {
      // Simulate RETURNING id for the check-and-set pattern
      return { rows: [], rowCount: 1 };
    }
    return { rows: [], rowCount: 0 };
  },

  async connect() {
    // Return a mock client
    return {
      query: this.query.bind(this),
      release() {},
    };
  },

  _setOrder(order) {
    this._rows.orders.set(order.id, order);
  },

  _reset() {
    this._rows.orders.clear();
    this._rows.outbox = [];
    this._rows.webhook_events.clear();
  },
};
```

This is not a full Postgres emulator. It is just enough to let your unit tests run without spinning up a real database. For integration tests you want the real thing; for unit tests a stub is fine.

If writing this fake pool feels like a lot of work, it is -- that is the price of thorough unit tests. The alternative is a full Postgres test database per CI run (slower, harder to set up, more flaky). Pick your trade.

---

## The Idempotency Test

The most important test to write. Write it first and keep it passing forever:

```javascript
test("duplicate callbacks do not double-process", async () => {
  fakePool._reset();
  fakePool._setOrder({
    id: "o1",
    payment_status: "initiated",
    mpesa_checkout_request_id: "CO_TEST",
    subtotal_cents: 10000,
  });

  const callback = {
    Body: {
      stkCallback: {
        CheckoutRequestID: "CO_TEST",
        ResultCode: 0,
        CallbackMetadata: { Item: [
          { Name: "Amount", Value: 100 },
          { Name: "MpesaReceiptNumber", Value: "N123" },
        ]},
      },
    },
  };

  // Fire twice in parallel
  await Promise.all([
    payments.handleCallback("mpesa", callback),
    payments.handleCallback("mpesa", callback),
  ]);

  // Exactly one outbox entry
  assert.equal(fakePool._rows.outbox.length, 1);
});
```

This test catches every regression you could introduce in the idempotency code. Run it often.

---

## Integration Test Against The Real Sandbox

One real test per provider, gated behind an env var so CI does not run them unless you opt in:

```javascript
// test/integration/mpesa.integration.test.js
const { test, skip } = require("node:test");

if (!process.env.MPESA_INTEGRATION === "true") {
  console.log("Skipping M-Pesa integration tests");
  process.exit(0);
}

test("real STK push to sandbox", async () => {
  const result = await payments.initiate("mpesa", {
    orderId: "integration-test",
    phone: "254708374149",
    amountCents: 100,
  });
  assert.ok(result.reference);
  // This leaves an initiated-but-never-completed order in the test DB. Accept that.
});
```

Run with:

```bash
MPESA_INTEGRATION=true npm test
```

---

## Test Coverage

Measure with `c8` (a lightweight built-in-to-Node coverage tool):

```bash
npm install --save-dev c8
```

```json
// package.json
"scripts": {
  "test": "node --test test/",
  "test:coverage": "c8 node --test test/"
}
```

Aim for 70% coverage. Not 100 -- going from 80 to 100 takes longer than going from 0 to 80 and the last 20% are almost always low-value cases. 70% is the sweet spot.

---

## Checkpoint

1. `npm test` runs all unit tests in under 5 seconds.
2. The idempotency test passes and fails in meaningful ways when you deliberately break the handler.
3. `npm run test:coverage` prints a coverage report showing >=70% for `src/`.
4. An integration test with `MPESA_INTEGRATION=true` runs against the sandbox.
5. CI runs unit tests on every push but skips integration by default.

Commit:

```bash
git add .
git commit -m "test: unit tests fake providers and idempotency coverage"
```

---

## What You Learned

- Unit tests use fakes; integration tests use sandboxes.
- Injectable provider registries make unit tests trivial.
- A minimal fake pool is worth the investment for test speed.
- The idempotency test is the first one you should write.
- 70% coverage is enough.

Tomorrow: publishing.
