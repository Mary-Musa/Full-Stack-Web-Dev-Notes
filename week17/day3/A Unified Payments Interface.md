# Week 17, Day 3: A Unified Payments Interface

By the end of today, your Next.js shop calls one function for any payment provider. `createOrder` does not know whether M-Pesa, Airtel, or Stripe is handling the money. The payment provider is a swappable implementation behind a single interface. This is the first real piece of the "reusable Payments Service" deliverable from the Phase 3 roadmap.

**Prior-week concepts you will use today:**
- M-Pesa (Week 16), Airtel (Week 17 Day 2), Stripe (Week 17 Day 1)
- Service layer patterns (Week 12, Day 2)
- Interface segregation -- one shape, many implementations

**Estimated time:** 3 hours

---

## The Interface

Every payment provider can do three things in our shop:

1. **Initiate** a payment for an order and return a "next step" the customer can act on.
2. **Receive a callback** from the provider when the payment succeeds or fails.
3. **Query** the provider for the latest state of a payment (reconciliation).

Express that as a plain-JS interface -- a shared shape every provider must match:

```javascript
// server/services/payments/provider.js

/**
 * The shape every payment provider must implement.
 *
 * initiate({ orderId, phone, amountCents, metadata }) ->
 *   { method, reference, nextStep }
 *
 * - method: "mpesa" | "airtel" | "stripe"
 * - reference: string the provider calls back with
 * - nextStep: { type: "wait" } for push-based,
 *             { type: "redirect", url } for hosted checkout
 *
 * handleCallback(body) -> void
 *   Processes an incoming webhook. Idempotent.
 *
 * queryStatus(reference) ->
 *   { status: "pending" | "success" | "failed" | "cancelled" }
 */
```

JavaScript has no type-level interfaces (without TypeScript), so the interface lives in a comment and the discipline lives in code review. In TypeScript we would write this as `interface PaymentProvider { ... }`. In plain JS the test is: every provider module exports the same three functions with the same signatures.

---

## Three Providers Behind One Door

Create `server/services/payments/index.js`:

```javascript
// server/services/payments/index.js
const mpesa = require("./mpesaProvider");
const airtel = require("./airtelProvider");
const stripe = require("./stripeProvider");

const PROVIDERS = {
  mpesa,
  airtel,
  stripe,
};

function getProvider(method) {
  const provider = PROVIDERS[method];
  if (!provider) throw new Error(`Unknown payment method: ${method}`);
  return provider;
}

async function initiate(method, payload) {
  return getProvider(method).initiate(payload);
}

async function handleCallback(method, body) {
  return getProvider(method).handleCallback(body);
}

async function queryStatus(method, reference) {
  return getProvider(method).queryStatus(reference);
}

module.exports = { initiate, handleCallback, queryStatus };
```

Three functions. The shop calls `initiate("mpesa", {...})` or `initiate("stripe", {...})` and does not care which file runs. The registry lives in one place (`PROVIDERS`), adding a new provider means writing the module and adding a line.

Create the three provider modules, each wrapping your existing code:

```javascript
// server/services/payments/mpesaProvider.js
const legacy = require("../payments.service"); // existing Week 16 code

async function initiate({ orderId, phone, amountCents }) {
  const { checkoutRequestId } = await legacy.initiateStkPush({ orderId, phone, amount: amountCents });
  return {
    method: "mpesa",
    reference: checkoutRequestId,
    nextStep: { type: "wait" },
  };
}

async function handleCallback(body) {
  return legacy.handleCallback(body);
}

async function queryStatus(reference) {
  const order = await legacy.getStatus(reference);
  return { status: order?.payment_status || "pending" };
}

module.exports = { initiate, handleCallback, queryStatus };
```

```javascript
// server/services/payments/airtelProvider.js
const legacy = require("../airtel.service");

async function initiate({ orderId, phone, amountCents }) {
  const { referenceId } = await legacy.initiateAirtelPayment({ orderId, phone, amountCents });
  return {
    method: "airtel",
    reference: referenceId,
    nextStep: { type: "wait" },
  };
}

async function handleCallback(body) {
  return legacy.handleAirtelCallback(body);
}

async function queryStatus(reference) {
  const result = await legacy.queryAirtelStatus(reference);
  return { status: mapAirtelStatus(result?.status?.code) };
}

function mapAirtelStatus(code) {
  if (code === "TS") return "success";
  if (code === "TF") return "failed";
  return "pending";
}

module.exports = { initiate, handleCallback, queryStatus };
```

```javascript
// server/services/payments/stripeProvider.js
const Stripe = require("stripe");
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);
const { query } = require("../../config/db");

async function initiate({ orderId, amountCents, metadata }) {
  // Note: the Next.js shop already creates the Stripe session directly
  // (Week 17 Day 1). Here we expose the same call via the Express server
  // so the interface is uniform across providers.

  const session = await stripe.checkout.sessions.create({
    mode: "payment",
    payment_method_types: ["card"],
    line_items: metadata.lineItems,
    success_url: metadata.successUrl,
    cancel_url: metadata.cancelUrl,
    metadata: { order_id: orderId },
  });

  await query(
    "UPDATE orders SET stripe_session_id = $1 WHERE id = $2",
    [session.id, orderId]
  );

  return {
    method: "stripe",
    reference: session.id,
    nextStep: { type: "redirect", url: session.url },
  };
}

async function handleCallback(body) {
  // Stripe webhooks use a different entry point (Next.js API route) and
  // a signed body. For symmetry we expose a no-op here.
  console.log("stripeProvider.handleCallback is handled in Next.js");
}

async function queryStatus(reference) {
  const session = await stripe.checkout.sessions.retrieve(reference);
  const status = session.payment_status === "paid" ? "success" : "pending";
  return { status };
}

module.exports = { initiate, handleCallback, queryStatus };
```

---

## The Unified Route

Replace the three scattered payment endpoints with one:

```javascript
// server/routes/payments.routes.js
const express = require("express");
const router = express.Router();
const payments = require("../services/payments");
const asyncHandler = require("../middleware/asyncHandler");

router.post("/initiate", asyncHandler(async (req, res) => {
  const { method, orderId, phone, amountCents, metadata } = req.body;
  const result = await payments.initiate(method, { orderId, phone, amountCents, metadata });
  res.json(result);
}));

router.post("/callback/:method", express.json(), asyncHandler(async (req, res) => {
  // Respond fast, process after
  res.json({ received: true });
  payments.handleCallback(req.params.method, req.body).catch((err) =>
    console.error(`${req.params.method} callback error:`, err)
  );
}));

router.get("/status", asyncHandler(async (req, res) => {
  const { method, reference } = req.query;
  const result = await payments.queryStatus(method, reference);
  res.json(result);
}));

module.exports = router;
```

Three endpoints instead of nine. The provider is a path param (`/callback/mpesa`, `/callback/airtel`) so Daraja and Airtel each call their own URL. A query param for status (`?method=mpesa&reference=ws_CO_...`) keeps the status endpoint simple.

### Redirect the legacy routes

The Week 16 `/api/payments/stk` route still exists. Keep it for backwards compatibility but have it call through the unified interface under the hood:

```javascript
router.post("/stk", asyncHandler(async (req, res) => {
  const { orderId, phone, amount } = req.body;
  const result = await payments.initiate("mpesa", { orderId, phone, amountCents: amount });
  res.json({ checkoutRequestId: result.reference });
}));
```

Deprecated-but-working. Week 18 we remove the old routes once the Next.js shop is migrated.

---

## The Next.js Side

Update `createOrder.js` to use the unified endpoint:

```javascript
// after committing the order:
if (paymentMethod !== "cod") {
  const metadata = paymentMethod === "stripe"
    ? {
        lineItems: items.map((i) => {
          const p = productMap[i.id];
          return {
            price_data: {
              currency: "kes",
              product_data: { name: p.name },
              unit_amount: p.price_cents,
            },
            quantity: i.quantity,
          };
        }),
        successUrl: `${process.env.NEXT_PUBLIC_BASE_URL}/checkout/confirmed`,
        cancelUrl: `${process.env.NEXT_PUBLIC_BASE_URL}/checkout/cancelled`,
      }
    : {};

  const initRes = await fetch(`${process.env.CRM_SERVER_URL}/api/payments/initiate`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      method: paymentMethod,
      orderId,
      phone: customer.phone,
      amountCents: subtotalCents,
      metadata,
    }),
  });
  const initData = await initRes.json();
  if (!initRes.ok) return { orderId, error: "Payment init failed" };

  return {
    orderId,
    reference: initData.reference,
    nextStep: initData.nextStep,
  };
}
```

The client-side Pay button reads `nextStep`:

```jsx
const result = await createOrder(...);
if (result.nextStep?.type === "redirect") {
  window.location.href = result.nextStep.url;
} else if (result.nextStep?.type === "wait") {
  router.push("/checkout/awaiting-payment");
}
```

One code path, three providers. Clean.

---

## Why This Matters

Before today the shop had three hard-coded branches in `createOrder`: if M-Pesa, call X; if Airtel, call Y; if Stripe, call Z. That works for three providers. It does not scale to five or ten -- and it makes adding a new provider (Paystack, Flutterwave, MTN MoMo) a surgical operation across many files.

With the unified interface, adding a new provider means:

1. Write `server/services/payments/paystackProvider.js` with `initiate`, `handleCallback`, `queryStatus`.
2. Register it in `PROVIDERS` in `index.js`.
3. Add the provider's webhook URL to their dashboard pointing at `/api/payments/callback/paystack`.
4. Add a radio option in the payment UI.

Four changes. None of them touch the Next.js shop's business logic. That is the payoff for the abstraction.

---

## Testing With All Three

Run through a full purchase with each provider:

1. **M-Pesa** -- same sandbox flow from Week 16.
2. **Airtel** -- Airtel sandbox number.
3. **Stripe** -- test card `4242...`.

All three should:
- Create an order with the correct `payment_method`.
- Transition to `paid` after the webhook.
- Fire a WhatsApp confirmation.
- Appear identically in the admin dashboard (filter by `payment_method`).

The admin dashboard should show the provider-specific references (`mpesa_checkout_request_id`, `airtel_reference`, `stripe_session_id`) on the order detail page. One schema, three keys.

---

## Checkpoint

1. `server/services/payments/index.js` has three registered providers.
2. `POST /api/payments/initiate` with `method: "mpesa"` works. With `method: "airtel"` works. With `method: "stripe"` works.
3. `POST /api/payments/callback/mpesa` routes to the M-Pesa handler.
4. `GET /api/payments/status?method=mpesa&reference=ws_CO_...` returns a status.
5. The Next.js checkout page has no provider-specific code in `createOrder` -- it just calls the unified endpoint.
6. Adding a fake fourth provider (`fakeProvider.js` that always returns success) works by registering it and nothing else changes.
7. Legacy routes (`/api/payments/stk`) still work for backwards compatibility but are marked deprecated in a comment.

Commit:

```bash
git add .
git commit -m "refactor: unified payments interface behind three providers"
```

---

## What You Learned

- Interface segregation: one shape, many implementations.
- The shop should not know which provider is handling a payment.
- Adding a new provider becomes a local change, not a cross-cutting refactor.
- Deprecation compatibility: keep old routes working while you migrate.

Tomorrow we add provider selection UX -- letting the customer pick the right provider based on their phone number, location, and preferences -- plus smart fallbacks when a provider is down.
