# Week 19, Day 2: Public API and README

By the end of today, `mctaba-payments` has a concise public API documented in a README a stranger could read and understand in ten minutes. Every exported function has a docstring. The README has a 30-second quickstart. Config options are listed with their types and defaults.

**Prior concepts:** yesterday's extraction.

**Estimated time:** 2 hours. Less than a full build day -- documentation, not code.

---

## What A README For A Library Looks Like

A good library README has exactly four sections, in this order:

1. **The 30-second pitch** -- one sentence, one code block.
2. **Install and init** -- two commands plus one config snippet.
3. **The API reference** -- every public function with arguments and return shape.
4. **Everything else** -- troubleshooting, security, contributing.

People read top-down and stop when their question is answered. Put the quick answer first.

---

## The README

Create `mctaba-payments/README.md`:

```markdown
# mctaba-payments

A minimal Node package for taking payments in Kenya. Supports M-Pesa STK Push, Airtel Money, and Stripe Checkout. Designed for Node 20+ with a Postgres database.

## Quick start

```bash
npm install mctaba-payments
```

```javascript
const { Pool } = require("pg");
const mctabaPayments = require("mctaba-payments");

const payments = mctabaPayments.init({
  pool: new Pool({ connectionString: process.env.DATABASE_URL }),
  config: {
    mpesa: { consumerKey: "...", consumerSecret: "...", shortcode: "174379", passkey: "..." },
    stripe: { secretKey: "sk_test_...", webhookSecret: "whsec_..." },
  },
});

// Initiate a payment:
const result = await payments.initiate("mpesa", {
  orderId: "order-123",
  phone: "+254712345678",
  amountCents: 250000, // KSh 2,500
});
console.log(result.reference);
```

That is it. Add callback routes to your web server and you are taking payments.

## Install & setup

### Requirements

- Node 20 or later
- Postgres 14 or later
- Sandbox or live credentials for M-Pesa Daraja and/or Stripe and/or Airtel

### Database schema

The package expects these tables in your database. Run them once (or use migrations):

```sql
CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id TEXT NOT NULL,
  method TEXT NOT NULL,
  reference TEXT,
  amount_cents INTEGER NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE outbox (
  id BIGSERIAL PRIMARY KEY,
  event_type TEXT NOT NULL,
  payload JSONB NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  processed_at TIMESTAMPTZ,
  attempts INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE webhook_events (id TEXT PRIMARY KEY, received_at TIMESTAMPTZ NOT NULL);
CREATE TABLE refunds (...);
CREATE TABLE reconciliation_runs (...);
CREATE TABLE reconciliation_mismatches (...);
```

A full schema is in [`sql/schema.sql`](sql/schema.sql) in this repo. Copy it or merge with your own migrations.

### Initialisation

```javascript
const payments = require("mctaba-payments").init({
  pool,    // required: your pg Pool
  config,  // required: provider config (see below)
});
```

`init` throws if the config is invalid, so missing keys fail loudly.

## Configuration

```javascript
{
  mpesa: {
    consumerKey: string,
    consumerSecret: string,
    shortcode: string,
    passkey: string,
    callbackUrl: string,
    env: "sandbox" | "production"    // default: "sandbox"
  },
  airtel: {
    clientId: string,
    clientSecret: string,
    country: string,                 // default: "KE"
    currency: string                 // default: "KES"
  },
  stripe: {
    secretKey: string,
    webhookSecret: string,
  }
}
```

Each provider block is optional. If omitted, attempts to use that provider throw.

## API

### `init(options) -> Payments`

Initialises the service with the given pool and config. Returns the public API object.

### `payments.initiate(method, args) -> Promise<InitiateResult>`

Initiates a payment.

- `method`: `"mpesa" | "airtel" | "stripe"`
- `args`: `{ orderId, phone, amountCents, metadata? }`
- Returns: `{ reference: string, nextStep: { type: "wait" | "redirect", url? } }`

### `payments.handleCallback(method, body, headers?) -> Promise<void>`

Processes an incoming webhook. Idempotent. Does not throw on duplicates.

- `method`: `"mpesa" | "airtel" | "stripe"`
- `body`: the raw request body (string for Stripe, parsed JSON for others)
- `headers`: request headers, used for signature verification

### `payments.verifyWebhook(method, body, headers) -> boolean`

Returns true if the webhook signature is valid. Call this in your route handler before `handleCallback`.

### `payments.refund({ orderId, amountCents, reason, method }) -> Promise<RefundResult>`

Creates a refund. Throws if the order has already been refunded in full.

### `payments.queryStatus(method, reference) -> Promise<{ status }>`

Polls the provider for the current state of a payment.

### `payments.reconcile(date) -> Promise<ReconciliationResult>`

Runs reconciliation for a given date. Returns a summary of mismatches found.

## Webhook routes

The package does not provide HTTP routes -- you do. Here is a minimal Express example:

```javascript
app.post("/api/payments/callback/:method", express.json(), async (req, res) => {
  res.json({ received: true });
  try {
    const valid = payments.verifyWebhook(req.params.method, JSON.stringify(req.body), req.headers);
    if (!valid) return;
    await payments.handleCallback(req.params.method, req.body);
  } catch (err) {
    console.error("webhook error:", err);
  }
});
```

For Stripe, verify against the raw body, not the parsed JSON:

```javascript
app.post("/api/payments/callback/stripe",
  express.raw({ type: "application/json" }),
  async (req, res) => {
    res.json({ received: true });
    try {
      await payments.handleCallback("stripe", req.body, req.headers);
    } catch (err) { /* log */ }
  }
);
```

## Troubleshooting

- **"Invalid signature"** -- your `webhookSecret` does not match Stripe's. Double-check you are using the right secret for the right environment.
- **"STK push rejected"** -- invalid Daraja credentials or wrong shortcode. Verify with Safaricom's portal.
- **Airtel token expires constantly** -- make sure you are caching with a buffer (the package does this automatically; check you are not bypassing it).

## Security

See [SECURITY.md](SECURITY.md) for the full threat model and what the package protects against. Short version: signature verification, IP allow-listing for M-Pesa, idempotency via event ids, rate limiting is *your* responsibility.

## License

MIT.
```

---

## Docstrings Inside The Code

Every exported function gets a JSDoc comment:

```javascript
/**
 * Initiates a payment through the specified provider.
 *
 * @param {string} method - "mpesa" | "airtel" | "stripe"
 * @param {Object} args
 * @param {string} args.orderId - Your internal order id. Echoed back in the callback.
 * @param {string} args.phone - Customer phone in international format (+2547...)
 * @param {number} args.amountCents - Amount in cents (KSh 100 = 10000)
 * @param {Object} [args.metadata] - Provider-specific metadata (required for Stripe).
 * @returns {Promise<{ reference: string, nextStep: { type: string, url?: string } }>}
 * @throws if the config is missing, amount is invalid, or provider rejects.
 */
async function initiate(method, args) { /* ... */ }
```

Even in a plain JS project, JSDoc gives editors type hints and readers a reference. Keep them short -- one-liner description, parameters, return, errors.

---

## A `SECURITY.md` File

```markdown
# Security

mctaba-payments handles money, which means hostile traffic is a certainty. Here is what the package does and does not protect against.

## What the package does

- Verifies Stripe webhook signatures against the raw body.
- Verifies Airtel webhook signatures when provided.
- Idempotency via event-id dedup tables -- duplicate callbacks are no-ops.
- Refund limits -- the package refuses to refund more than the original amount.
- Input validation at `initiate` -- amounts, phone numbers, allowed methods.

## What the package does NOT do (your responsibility)

- **Rate limiting** on your webhook endpoints. Use `express-rate-limit` or similar.
- **IP allow-listing** of M-Pesa callbacks. The package provides `verifyIp(method, ip)` -- you must call it.
- **Authentication** of your admin routes. The package's reconciliation interface is not auth-aware.
- **Secrets management**. Read env vars yourself; pass the config object in.
- **Database backups**. The package writes to your database; your database's reliability is yours.

## Reporting

Found a security issue? Email `security@mctaba.co.ke`. Do not file a public GitHub issue.

## Dependencies

Dependencies are pinned. Run `npm audit` monthly.
```

---

## Checkpoint

1. `README.md` starts with a 30-second quickstart block that compiles and runs.
2. Every exported function has a JSDoc.
3. `SECURITY.md` exists and lists what is and is not the package's responsibility.
4. Reading only the README, someone not on the team could install, configure, and take their first test payment in under 20 minutes.
5. Running `npx jsdoc -r src/` produces HTML docs (optional tooling check).

Commit:

```bash
cd mctaba-payments
git add .
git commit -m "docs: add readme security and jsdoc"
```

---

## What You Learned

- A library README has four sections in a specific order.
- JSDoc gives editor hints even in plain JS.
- `SECURITY.md` makes the threat model explicit.
- Good docs are shorter than bad docs.

Tomorrow: tests.
