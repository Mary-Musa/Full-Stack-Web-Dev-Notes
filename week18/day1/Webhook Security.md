# Week 18, Day 1: Webhook Security

> **AI boundaries this week:** 45% manual / 55% AI (SECURITY DIP). Habit: *Audit every security line AI writes -- line-by-line.* Money + auth rules both apply. See [ai.md](../ai.md).

By the end of today, every webhook endpoint in your app verifies the request came from who it claims to come from. Stripe events are HMAC-verified, M-Pesa callbacks are IP-allowlisted and signature-checked where possible, Airtel is verified via their provided signature. Any request that fails verification is rejected with 401 and never touches your database.

This is the day you stop trusting the public internet.

**Prior-week concepts you will use today:**
- Stripe webhooks with signature verification (Week 17, Day 1)
- M-Pesa and Airtel webhook handlers (Week 16 and Week 17)
- HMAC basics from WhatsApp (Week 11, Day 2)

**Estimated time:** 3 hours

---

## The Week Ahead

| Day | Focus |
|---|---|
| Day 1 (today) | Webhook signature verification, IP allow-listing, request logging. |
| Day 2 | Idempotency at every layer (retries, duplicate clicks, out-of-order callbacks). |
| Day 3 | Refunds: Daraja B2C, Stripe refunds, Airtel reversal. |
| Day 4 | Reconciliation: daily sweep, mismatch detection, admin tools. |
| Day 5 | Recap. |
| Weekend | Payment layer hardening on your shop. No new features -- just safety. |

---

## What Could Actually Go Wrong

If your payment webhook is not verified, any attacker who knows the URL can POST a fake callback and flip orders to `paid`. You give them free products.

If your webhook is verified but not idempotent, a provider retry can cause a double-charge or a double-ship. You lose money twice: once on the lost inventory and once on the refund you have to issue.

If your refund logic is wrong, you refund the wrong order, or you refund twice, or the refund silently fails and the customer calls angry.

If there is no reconciliation, money can go missing for weeks before anyone notices. Safaricom's records say "paid"; your database says "failed". The customer is angry and the owner is baffled.

Every bullet here has happened to real shops. This week is how we prevent them.

---

## Stripe Webhook Verification (Review)

You already did this in Week 17 Day 1:

```javascript
const event = stripe.webhooks.constructEvent(rawBody, signature, WEBHOOK_SECRET);
```

The signature scheme: Stripe computes an HMAC-SHA256 of the request body using your webhook secret, and sends it in the `Stripe-Signature` header. `constructEvent` re-computes the HMAC and compares. Constant-time comparison, safe against timing attacks.

Two things to tighten:

**1. Do not skip signature verification in any environment.** Not in dev, not in staging, not in tests. In dev you use the Stripe CLI's forwarded secret. Never `if (process.env.NODE_ENV !== 'production') skipVerification()`. That `if` will eventually ship.

**2. Use the raw body, not the parsed JSON.** `await req.text()` gives you the raw string. Once JSON-parsed and re-stringified, whitespace changes break the HMAC. Always verify against the raw body.

---

## M-Pesa Signature (Not Really)

Daraja does **not** sign callbacks. That is a known weakness of the M-Pesa API. Safaricom's position is that the callback URL is private and secret, which is... not great.

Defenses you can layer:

**1. IP allow-list.** Safaricom publishes the IP ranges their callbacks come from. Hard-code them and reject callbacks from any other IP. Their current ranges (subject to change -- check Daraja docs):

```javascript
const SAFARICOM_IPS = [
  "196.201.214.200/29",
  "196.201.213.114/32",
  // ... full list from Daraja docs
];

function isFromSafaricom(ip) {
  // Use a CIDR matching library like `ip-range-check`
  const ipRangeCheck = require("ip-range-check");
  return ipRangeCheck(ip, SAFARICOM_IPS);
}

// In the callback handler:
const ip = req.headers["x-forwarded-for"]?.split(",")[0] || req.connection.remoteAddress;
if (!isFromSafaricom(ip)) {
  console.warn("M-Pesa callback from non-Safaricom IP:", ip);
  return res.status(403).json({ error: "Forbidden" });
}
```

**2. Obscure the URL.** Register the callback under a path with a random component: `/api/payments/callback/mpesa-a8f3k2n9l5`. The random token acts as a second-factor. Generate it once, put it in `.env`, never commit.

**3. Amount + reference matching** (you already do this in Week 16). Even if a fake callback gets through, it can only flip an order whose `CheckoutRequestID` the attacker already knows -- and that ID was never sent to them.

Together these three are enough. Perfect would be HMAC, but Daraja does not give you one.

---

## Airtel Signature

Airtel signs callbacks but their documentation on the exact scheme is sparse. Current approach:

```javascript
const crypto = require("crypto");

function verifyAirtelSignature(body, signature) {
  const computed = crypto
    .createHmac("sha256", process.env.AIRTEL_CALLBACK_SECRET)
    .update(JSON.stringify(body))
    .digest("hex");
  return crypto.timingSafeEqual(
    Buffer.from(computed),
    Buffer.from(signature || "")
  );
}
```

Note `timingSafeEqual` -- always use it, never `===`, when comparing secrets. A naive comparison leaks information through execution time.

Where does `AIRTEL_CALLBACK_SECRET` come from? You set it when registering the callback URL with Airtel. If they do not give you an option to set one, fall back to IP allow-listing plus the reference-matching defence.

---

## A General Webhook Middleware

Build one middleware that every payment webhook runs through:

```javascript
// server/middleware/webhookVerifier.js
const ipRangeCheck = require("ip-range-check");

const PROVIDERS = {
  mpesa: {
    allowedIps: ["196.201.214.200/29", "196.201.213.114/32"],
    requireSignature: false,
  },
  airtel: {
    allowedIps: null,
    requireSignature: true,
    verify(body, headers) {
      const sig = headers["x-signature"];
      // ... the verification function above
    },
  },
  // Stripe verifies inside its own Next.js route; no middleware needed here.
};

function webhookVerifier(providerName) {
  return (req, res, next) => {
    const provider = PROVIDERS[providerName];
    if (!provider) return next(new Error(`Unknown webhook provider: ${providerName}`));

    if (provider.allowedIps) {
      const ip = req.headers["x-forwarded-for"]?.split(",")[0] || req.connection.remoteAddress;
      if (!ipRangeCheck(ip, provider.allowedIps)) {
        console.warn(`${providerName} webhook from unauthorized IP:`, ip);
        return res.status(403).json({ error: "Forbidden" });
      }
    }

    if (provider.requireSignature) {
      if (!provider.verify(req.body, req.headers)) {
        return res.status(401).json({ error: "Invalid signature" });
      }
    }

    next();
  };
}

module.exports = webhookVerifier;
```

Apply:

```javascript
router.post("/callback/:method", webhookVerifier(":method"), handler);
```

You can get fancier with dynamic middleware injection; the above is simple and explicit.

---

## Structured Logging For Every Webhook

Logging webhooks is easy to do badly. Either you log nothing and debugging is impossible, or you log everything and secrets end up in your log files.

The right middle:

```javascript
async function logWebhook(provider, body, outcome) {
  await query(
    `INSERT INTO webhook_log (provider, reference, result_code, received_at, ip, processed)
     VALUES ($1, $2, $3, NOW(), $4, $5)`,
    [provider, body?.reference || body?.stkCallback?.CheckoutRequestID, body?.ResultCode || body?.status_code, req.ip, outcome]
  );
}
```

Schema:

```sql
CREATE TABLE webhook_log (
  id BIGSERIAL PRIMARY KEY,
  provider TEXT NOT NULL,
  reference TEXT,
  result_code TEXT,
  received_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  ip INET,
  processed TEXT
);
CREATE INDEX idx_webhook_log_received ON webhook_log(received_at DESC);
CREATE INDEX idx_webhook_log_provider_ref ON webhook_log(provider, reference);
```

Log the provider, the reference (so you can find specific events), the result code, the IP, and the outcome ("processed" / "skipped_duplicate" / "error"). Do not log the full body -- a body can contain a phone number, and phone numbers are PII you probably do not want in plaintext.

When a customer calls angry, you `SELECT * FROM webhook_log WHERE reference = 'ws_CO_...'` and see every attempt in order.

---

## Rate Limiting

A webhook endpoint is public and a target for abuse. Add a basic rate limit:

```bash
npm install express-rate-limit
```

```javascript
const rateLimit = require("express-rate-limit");

const webhookLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 60,             // 60 requests per minute per IP
  message: { error: "Too many webhooks" },
});

router.post("/callback/:method", webhookLimiter, webhookVerifier(":method"), handler);
```

This is a blunt instrument. Legitimate webhooks rarely exceed this rate. A flood of junk traffic hits the limit in seconds and the attacker moves on. For precise rate limiting tied to the provider, use a Redis-backed limiter in Week 23 when you have BullMQ.

---

## Admin View: Recent Webhooks

Build a small admin page that shows the last 50 webhook events:

```jsx
// app/admin/webhooks/page.js
import { query } from "@/lib/db";
import { requireAdmin } from "@/app/actions/auth";

export default async function WebhooksPage() {
  await requireAdmin();
  const { rows } = await query(
    `SELECT * FROM webhook_log ORDER BY received_at DESC LIMIT 50`
  );
  return (
    <div className="max-w-4xl mx-auto p-8">
      <h1 className="text-2xl font-bold mb-6">Recent Webhooks</h1>
      <table className="w-full text-sm">
        <thead>
          <tr><th>Time</th><th>Provider</th><th>Reference</th><th>Result</th><th>Outcome</th></tr>
        </thead>
        <tbody>
          {rows.map((r) => (
            <tr key={r.id} className="border-t">
              <td className="py-2">{new Date(r.received_at).toLocaleString("en-KE")}</td>
              <td>{r.provider}</td>
              <td className="font-mono text-xs">{r.reference?.slice(0, 16)}</td>
              <td>{r.result_code}</td>
              <td>{r.processed}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

Two seconds to build, saves hours of debugging.

---

## Checkpoint

1. Sending a fake Stripe webhook (random HMAC) returns 400 and does not write to the database.
2. Sending a fake M-Pesa callback from a non-Safaricom IP returns 403. (You can test by using curl from your laptop and it will fail.)
3. `webhook_log` table has rows for every real callback with the provider, reference, and outcome.
4. `/admin/webhooks` shows the last 50.
5. Flooding the endpoint with 100 requests/minute triggers the rate limiter and returns 429.
6. Timing test: computing the correct signature takes the same time as computing an incorrect one, because you used `timingSafeEqual`.

Commit:

```bash
git add .
git commit -m "feat: webhook signature verification ip allow-listing and logging"
```

---

## What You Learned

- Signature verification must use raw body and constant-time comparison.
- M-Pesa is unsigned; defend with IP allow-list plus obscure URL plus reference matching.
- Log webhooks to a dedicated table, not the application log.
- Rate-limit public endpoints with a simple bucket.
- Admin visibility into recent webhooks pays for itself the first time a customer calls.

Tomorrow we go deeper on idempotency. You have handled the easy case (duplicate Stripe events); tomorrow is the hard cases (out-of-order callbacks, retries across process restarts, partial failures mid-transaction).
