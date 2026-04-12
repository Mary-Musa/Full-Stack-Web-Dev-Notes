# Week 17, Day 2: Airtel Money Integration

By the end of today, the shop supports Airtel Money alongside M-Pesa and Stripe. The flow mirrors M-Pesa's: initiate a push, wait for the callback, mark the order paid. You will also see just how similar these APIs are under the surface -- and how the differences matter.

**Prior-week concepts you will use today:**
- The M-Pesa STK Push + callback flow (Week 16, Days 1-2)
- The Express payments service shape (Week 16, Day 1)
- Webhook idempotency patterns (Week 16, Day 2)

**Estimated time:** 3 hours

---

## Why Airtel Money

Airtel Money has roughly 15% of Kenya's mobile money market. It is the second-largest option after M-Pesa. For most customers M-Pesa is enough, but there are two constituencies where Airtel matters:

1. **Airtel-heavy regions** (some parts of northern and coastal Kenya where Airtel has better coverage than Safaricom).
2. **Price-sensitive customers** who have both SIMs and pick Airtel because its transaction fees are slightly lower.

Ignoring 15% of the market is a business decision. For a serious Kenyan shop, supporting both is standard.

---

## Setting Up Airtel Money Sandbox

Airtel's developer portal is https://developers.airtel.africa. Sign up, create a sandbox application, and get:

- `AIRTEL_CLIENT_ID`
- `AIRTEL_CLIENT_SECRET`
- `AIRTEL_X_COUNTRY=KE`
- `AIRTEL_X_CURRENCY=KES`

Add them to `server/.env`. Airtel's sandbox is less pleasant than Daraja's -- the docs are thinner and test numbers are harder to find. Pin the official sandbox tester numbers they give you on signup.

---

## The Auth Dance

Airtel uses OAuth2 client credentials flow -- a familiar pattern if you have done any API before. You exchange `client_id + client_secret` for an access token, cache it, and use it on the next call until it expires.

```javascript
// server/services/airtel.service.js
const axios = require("axios");
const env = require("../config/env");

const BASE_URL = "https://openapiuat.airtel.africa";
let cachedToken = null;
let cachedExpiry = 0;

async function getToken() {
  if (cachedToken && Date.now() < cachedExpiry) return cachedToken;

  const res = await axios.post(`${BASE_URL}/auth/oauth2/token`, {
    client_id: env.AIRTEL_CLIENT_ID,
    client_secret: env.AIRTEL_CLIENT_SECRET,
    grant_type: "client_credentials",
  });

  cachedToken = res.data.access_token;
  cachedExpiry = Date.now() + (res.data.expires_in - 60) * 1000; // refresh 1 min early
  return cachedToken;
}
```

The caching is important: Airtel rate-limits token requests heavily. If every payment fetched a new token you would be throttled within a few dozen orders. Cache for slightly less than the stated expiry so you never use a token past its deadline.

---

## Initiating a Payment

```javascript
async function initiateAirtelPayment({ orderId, phone, amountCents }) {
  const token = await getToken();
  const amountKsh = Math.ceil(amountCents / 100);
  const referenceId = `ord_${orderId.slice(0, 8)}_${Date.now()}`;

  const res = await axios.post(
    `${BASE_URL}/merchant/v1/payments/`,
    {
      reference: orderId.slice(0, 12),
      subscriber: {
        country: env.AIRTEL_X_COUNTRY,
        currency: env.AIRTEL_X_CURRENCY,
        msisdn: formatPhone(phone),
      },
      transaction: {
        amount: amountKsh,
        country: env.AIRTEL_X_COUNTRY,
        currency: env.AIRTEL_X_CURRENCY,
        id: referenceId,
      },
    },
    {
      headers: {
        Authorization: `Bearer ${token}`,
        "X-Country": env.AIRTEL_X_COUNTRY,
        "X-Currency": env.AIRTEL_X_CURRENCY,
      },
    }
  );

  if (res.data.status?.code !== "200") {
    throw new Error(`Airtel rejected: ${res.data.status?.message}`);
  }

  // Save the airtel reference for callback lookup
  await query(
    `UPDATE orders
     SET airtel_reference = $1, payment_status = 'initiated', updated_at = NOW()
     WHERE id = $2`,
    [referenceId, orderId]
  );

  return { referenceId };
}

function formatPhone(phone) {
  return phone.replace(/^\+/, "").replace(/^0/, "254");
}
```

Add the column:

```sql
ALTER TABLE orders ADD COLUMN airtel_reference TEXT;
CREATE INDEX idx_orders_airtel_ref ON orders(airtel_reference);
```

### How Airtel's flow differs from M-Pesa

- **Synchronous response.** Airtel returns a success/pending/failure code in the initial POST response. M-Pesa only tells you "accepted, wait for callback".
- **Push prompt.** Airtel sends a USSD popup (not a mobile app notification) to the customer's phone. They enter their Airtel PIN to approve.
- **Callback.** If the customer approves, Airtel sends a callback to your `/api/airtel/callback`. Same shape as any webhook.
- **Status polling.** Airtel also exposes a status-query endpoint you can poll if the callback never arrives. We use it for the same reason as M-Pesa's STK Push Query.

---

## The Callback Handler

```javascript
async function handleAirtelCallback(body) {
  // Airtel's callback body (simplified):
  // { transaction: { id, message, status_code, airtel_money_id }, hash: "..." }

  const txn = body?.transaction;
  if (!txn) return;

  const referenceId = txn.id;

  const { rows } = await query(
    "SELECT id, subtotal_cents, payment_status FROM orders WHERE airtel_reference = $1",
    [referenceId]
  );
  const order = rows[0];
  if (!order) {
    console.warn("Airtel callback for unknown reference:", referenceId);
    return;
  }

  if (order.payment_status === "success") {
    console.log("Duplicate Airtel callback, already processed");
    return;
  }

  // Airtel status codes:
  // TS -> Transaction Successful
  // TF -> Transaction Failed
  // TIP -> Transaction in Progress
  if (txn.status_code === "TS") {
    // Amount sanity check
    const expectedKsh = Math.ceil(order.subtotal_cents / 100);
    if (txn.amount && txn.amount !== expectedKsh) {
      console.error(`Airtel amount mismatch for ${order.id}`);
      await query(
        "UPDATE orders SET payment_status = 'failed' WHERE id = $1",
        [order.id]
      );
      return;
    }

    await query(
      `UPDATE orders
       SET payment_status = 'success', status = 'paid',
           airtel_transaction_id = $1, updated_at = NOW()
       WHERE id = $2`,
      [txn.airtel_money_id, order.id]
    );

    // Fire and forget WhatsApp
    orderNotifications.sendOrderConfirmation(order.id).catch(console.error);
  } else if (txn.status_code === "TF") {
    await query(
      "UPDATE orders SET payment_status = 'failed' WHERE id = $1",
      [order.id]
    );
  }
  // TIP -> leave as initiated, callback will come again
}
```

Add the column:

```sql
ALTER TABLE orders ADD COLUMN airtel_transaction_id TEXT;
```

### Signature verification

Airtel signs callbacks with an HMAC but the scheme is... quirky. The documentation flips between two methods depending on which page you read. The practical approach:

1. In production, talk to Airtel support and get explicit documentation for your endpoint.
2. For now, do basic checks: verify the callback comes from an Airtel IP range (Airtel publishes these), verify the `hash` field if present, and rely on `airtel_reference` being a non-guessable UUID prefix to make attacks hard.

This is not as clean as Stripe's signature verification, but Airtel's API is less mature. We do the best we can and double up with Week 18's idempotency and rate limiting.

---

## The Route

```javascript
// server/routes/airtel.routes.js
const express = require("express");
const router = express.Router();
const airtelService = require("../services/airtel.service");
const asyncHandler = require("../middleware/asyncHandler");

router.post("/initiate", asyncHandler(async (req, res) => {
  const { orderId, phone, amountCents } = req.body;
  const result = await airtelService.initiateAirtelPayment({ orderId, phone, amountCents });
  res.json(result);
}));

router.post("/callback", express.json(), asyncHandler(async (req, res) => {
  // Respond fast, process after
  res.json({ status: "received" });
  airtelService.handleAirtelCallback(req.body).catch((err) =>
    console.error("Airtel callback error:", err)
  );
}));

module.exports = router;
```

Wire in `index.js`:

```javascript
app.use("/api/airtel", require("./routes/airtel.routes"));
```

---

## Status Query Endpoint

Airtel has a transaction-status endpoint you can poll like M-Pesa's query:

```javascript
async function queryAirtelStatus(orderId) {
  const { rows } = await query("SELECT airtel_reference FROM orders WHERE id = $1", [orderId]);
  const ref = rows[0]?.airtel_reference;
  if (!ref) return { error: "No Airtel reference recorded" };

  const token = await getToken();
  const res = await axios.get(
    `${BASE_URL}/standard/v1/payments/${ref}`,
    {
      headers: {
        Authorization: `Bearer ${token}`,
        "X-Country": env.AIRTEL_X_COUNTRY,
        "X-Currency": env.AIRTEL_X_CURRENCY,
      },
    }
  );

  return res.data;
}
```

Admin dashboard gets a "Check Airtel status" button next to the "Check M-Pesa status" button. Symmetric UX.

---

## Updating The Next.js Flow

In `createOrder.js`, handle the `paymentMethod === "airtel"` branch:

```javascript
if (paymentMethod === "airtel") {
  try {
    const res = await fetch(`${process.env.CRM_SERVER_URL}/api/airtel/initiate`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        orderId,
        phone: customer.phone,
        amountCents: subtotalCents,
      }),
    });
    const data = await res.json();
    if (!res.ok) return { orderId, error: "Airtel init failed" };
    return { orderId, referenceId: data.referenceId };
  } catch (err) {
    return { orderId, error: "Could not reach Airtel service" };
  }
}
```

And the `AwaitingPaymentClient` polls the same status endpoint regardless of provider -- Day 3 we unify so the frontend stops caring which provider is in use.

---

## Checkpoint

1. Choosing Airtel on the checkout page initiates an Airtel push.
2. The sandbox test number receives a simulated USSD prompt.
3. Callback arrives at your Express webhook with `status_code: TS`.
4. Order flips to `paid`, `airtel_transaction_id` is stored.
5. Duplicate callback (POST the same body twice) does not double-process.
6. A `TF` (failed) callback leaves the order in `failed`.
7. Admin dashboard shows the Airtel reference on the order detail page.

Commit:

```bash
git add .
git commit -m "feat: airtel money integration with callback handling"
```

---

## What You Learned

- Airtel's flow mirrors M-Pesa's: initiate, wait for callback, verify amount, update status.
- OAuth2 client-credentials tokens must be cached with an expiry buffer.
- Airtel's signature verification is less clean than Stripe's; double up on other defences.
- Respond fast to webhook callbacks and process after.
- Status query endpoints save you when callbacks are missed.

Tomorrow we build a single Payments Service abstraction that makes M-Pesa, Airtel, and Stripe interchangeable. Your shop code stops caring which one it is calling.
