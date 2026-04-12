# Week 16, Day 4: Notifications, Retries, and Polish

By the end of today, Project 3 is feature-complete. Admins get notified of new orders, failed payments can be retried from both the customer page and the admin dashboard, and the full lifecycle of an order (`pending -> paid -> shipped -> delivered`) has a WhatsApp message at each transition. You also add a tiny "order confirmed" admin dashboard widget that makes the owner feel the business is alive.

**Prior-week concepts you will use today:**
- Yesterday's WhatsApp confirmation flow (Week 16, Day 3)
- The M-Pesa callback and idempotency (Week 16, Day 2)
- Admin state-machine transitions (Week 15, Day 4)

**Estimated time:** 3 hours

---

## Admin Gets Pinged On New Orders

Every time a new paid order lands, the shop owner wants to know in their pocket. Two cheap channels:

1. A WhatsApp message to the owner's phone.
2. An audible chime on the admin dashboard if it is open.

Let's wire the first; the second is 10 lines of `Audio()` at the end of the lesson.

### Notifying the owner

Add an env var `SHOP_OWNER_PHONE=+254712000000`. In `orderNotifications.service.js`:

```javascript
async function notifyOwnerOfNewOrder(orderId) {
  if (!env.SHOP_OWNER_PHONE) return;

  const { rows } = await query(
    `SELECT o.id, o.customer_name, o.subtotal_cents, o.delivery_address,
            COUNT(oi.id) AS item_count
     FROM orders o
     JOIN order_items oi ON oi.order_id = o.id
     WHERE o.id = $1
     GROUP BY o.id`,
    [orderId]
  );
  const order = rows[0];
  if (!order) return;

  const text = `New order ${order.id.slice(0, 8).toUpperCase()}
From: ${order.customer_name}
Items: ${order.item_count}
Total: KSh ${(order.subtotal_cents / 100).toLocaleString()}
Address: ${order.delivery_address}`;

  try {
    await whatsapp.sendTextMessage({ to: env.SHOP_OWNER_PHONE, text });
  } catch (err) {
    console.error("owner notification failed:", err);
  }
}
```

Call it right after the customer confirmation:

```javascript
// in payments.service.js handleCallback, success branch:
orderNotifications.sendOrderConfirmation(order.id).catch(console.error);
orderNotifications.notifyOwnerOfNewOrder(order.id).catch(console.error);
```

Two sends, both fire and forget. The owner has to be opted in (has messaged the test number in the last 24 hours) for session text to work on the sandbox. On production with a template, you pre-approve an `owner_new_order` template and bypass the opt-in rule.

### Admin dashboard chime

Add a tiny feature to the admin orders page: play a chime sound when a new order lands while the page is open. On the client:

```jsx
"use client";
import { useEffect, useRef, useState } from "react";

export default function NewOrderWatcher({ initialCount }) {
  const [count, setCount] = useState(initialCount);
  const audio = useRef(null);

  useEffect(() => {
    audio.current = new Audio("/chime.mp3");
    const interval = setInterval(async () => {
      const res = await fetch("/api/admin/order-count");
      const data = await res.json();
      if (data.count > count) {
        audio.current.play().catch(() => {});
        setCount(data.count);
      }
    }, 5000);
    return () => clearInterval(interval);
  }, [count]);

  return null;
}
```

Drop it on the admin orders page. Add a small API route `app/api/admin/order-count/route.js`:

```javascript
import { query } from "@/lib/db";
import { requireAdmin } from "@/app/actions/auth";

export async function GET() {
  await requireAdmin();
  const { rows } = await query("SELECT COUNT(*)::int AS count FROM orders WHERE status = 'paid'");
  return Response.json({ count: rows[0].count });
}
```

Put a `chime.mp3` in `public/`. Any short notification sound works -- keep it under 1 second. Modern browsers require user interaction before playing audio, so the chime will silently fail until the admin clicks somewhere on the page first. That is fine -- the admin clicks on every page.

This is the kind of tiny feature that makes real business owners feel their shop is real. A KSh 10 sound effect is worth more than half the code you wrote this week.

---

## Retrying Failed Payments

From the customer side, the "awaiting payment" page shows a "Retry" button when the status is `cancelled` or `failed`. Clicking it calls a new Server Action `retryPayment(orderId)` that triggers a new STK Push for the same order.

```javascript
// app/actions/payments.js
"use server";

export async function retryPayment(orderId) {
  const { rows } = await query(
    `SELECT id, customer_phone, subtotal_cents, payment_status
     FROM orders WHERE id = $1`,
    [orderId]
  );
  const order = rows[0];
  if (!order) return { error: "Order not found" };
  if (order.payment_status === "success") return { error: "Already paid" };

  try {
    const res = await fetch(`${process.env.CRM_SERVER_URL}/api/payments/stk`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        orderId: order.id,
        phone: order.customer_phone,
        amount: order.subtotal_cents,
      }),
    });
    const data = await res.json();
    if (!res.ok) return { error: "Payment service unavailable" };
    return { checkoutRequestId: data.checkoutRequestId };
  } catch (err) {
    return { error: "Could not retry payment" };
  }
}
```

The retry function is almost identical to the original STK push flow. The only difference: the order row already exists, so we do not create a new one. We reuse the row, generate a new `mpesa_checkout_request_id`, and the callback handler has no idea this is a retry -- it is just another callback for an existing order.

### Rate-limiting retries

A user clicking "Retry" fifty times should not fire fifty STK pushes. Add a server-side rate limit: reject retries within 30 seconds of the last attempt.

```javascript
// in payments.service.js initiateStkPush:
const { rows: existing } = await query(
  "SELECT updated_at FROM orders WHERE id = $1",
  [orderId]
);
const lastUpdate = new Date(existing[0].updated_at);
if (Date.now() - lastUpdate.getTime() < 30000) {
  throw new Error("Please wait 30 seconds before retrying");
}
```

This is not a perfect rate limit -- a determined attacker can still hit you. It is a polite rate limit that covers the "user clicked four times" case. Real rate limiting goes in Week 18 with webhook security.

---

## Lifecycle Messages: Paid, Shipped, Delivered

The customer gets a message at `paid`. They should also get one at `shipped` and `delivered`. Extend `updateOrderStatus` in `app/actions/orders.js`:

```javascript
// in the success branch of updateOrderStatus:
if (nextStatus === "shipped") {
  fetch(`${process.env.CRM_SERVER_URL}/api/orders/${orderId}/notify`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ event: "shipped" }),
  }).catch(console.error);
}
if (nextStatus === "delivered") {
  fetch(`${process.env.CRM_SERVER_URL}/api/orders/${orderId}/notify`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ event: "delivered" }),
  }).catch(console.error);
}
```

And on the Express side, a new endpoint:

```javascript
// server/controllers/orderNotifications.controller.js
async function notify(req, res) {
  const { event } = req.body;
  await orderNotifications.sendLifecycleUpdate(req.params.id, event);
  res.json({ ok: true });
}
```

In the service:

```javascript
async function sendLifecycleUpdate(orderId, event) {
  const { rows } = await query(
    `SELECT id, customer_name, customer_phone FROM orders WHERE id = $1`,
    [orderId]
  );
  const order = rows[0];
  if (!order) return;

  const firstName = order.customer_name.split(" ")[0];
  const shortId = order.id.slice(0, 8).toUpperCase();

  const messages = {
    shipped: `${firstName}, your order ${shortId} has been dispatched. It should reach you within a day.`,
    delivered: `${firstName}, your order ${shortId} was marked delivered. Thank you for shopping with us!`,
  };

  const text = messages[event];
  if (!text) return;

  await whatsapp.sendTextMessage({ to: order.customer_phone, text });
  await query(
    `INSERT INTO messages (direction, body, order_id) VALUES ('out', $1, $2)`,
    [text, orderId]
  );
}
```

Four messages total across the lifecycle:
1. Payment confirmation (from the M-Pesa callback)
2. Shipped (from the admin moving the order)
3. Delivered (from the admin or the delivery person)
4. Any reply from the customer (threaded back to the order via `context.id`)

This is a complete customer journey. The customer never downloads an app; every message is a reply to the first one, they all stack in the same WhatsApp thread, the shop owner can look up the conversation any time.

---

## Cash on Delivery Path

COD orders should not trigger M-Pesa. The Server Action already skips STK Push when `paymentMethod === "cod"`. But the confirmation flow is different:

- The order goes straight to `status = 'pending'` (awaiting payment on delivery).
- A confirmation WhatsApp fires immediately (no need to wait for a callback).
- The admin sees `payment_method = 'cod'` and knows to take cash on delivery.
- When the driver confirms delivery, the admin marks it `delivered` AND `paid` simultaneously.

Add a special transition for COD: `pending -> delivered` directly, which also marks `payment_status = 'success'`. The transition table grows:

```javascript
const TRANSITIONS = {
  pending: ["paid", "cancelled", "delivered"], // delivered directly for COD
  paid: ["shipped", "cancelled"],
  shipped: ["delivered"],
  delivered: [],
  cancelled: [],
};
```

And inside `updateOrderStatus`:

```javascript
if (current === "pending" && nextStatus === "delivered") {
  await query(
    `UPDATE orders SET status = 'delivered', payment_status = 'success', updated_at = NOW()
     WHERE id = $1 AND payment_method = 'cod'`,
    [orderId]
  );
}
```

Enforce that the shortcut only works for COD orders. A regular M-Pesa order moving from `pending` to `delivered` would skip the payment entirely -- a bug.

---

## The Empty-Dashboard Anti-Pattern

When the admin opens `/admin/orders` for the first time and sees an empty table, they think the shop is broken. Add a helpful empty state:

```jsx
{orders.length === 0 && (
  <div className="text-center py-16 text-gray-500">
    <p className="text-lg">No orders yet.</p>
    <p className="text-sm mt-2">Orders will appear here as customers check out.</p>
    <Link href="/products" className="mt-4 inline-block text-brand underline">
      View your storefront
    </Link>
  </div>
)}
```

Tiny change, big difference. Do this for every "list of things" page in the admin area -- products, customers, messages, whatever. An empty table is an accidental bug report.

---

## One More: The Public Order-Tracking Page

Customers who close the tab before the confirmation page should still be able to track their order. Without logins, the only identifier they have is the short id from their WhatsApp message.

Add `/track/[shortId]/page.js`:

```jsx
import { query } from "@/lib/db";
import { notFound } from "next/navigation";

export default async function TrackPage({ params }) {
  const { rows } = await query(
    `SELECT id, customer_name, status, payment_status, subtotal_cents, created_at
     FROM orders WHERE id::text LIKE $1 || '%'`,
    [params.shortId.toLowerCase()]
  );
  const order = rows[0];
  if (!order) notFound();

  return (
    <div className="max-w-md mx-auto p-8">
      <h1 className="text-2xl font-bold">Order {order.id.slice(0, 8).toUpperCase()}</h1>
      <p className="text-sm text-gray-500 mb-6">{new Date(order.created_at).toLocaleString("en-KE")}</p>
      <div className="space-y-2">
        <p>Customer: {order.customer_name}</p>
        <p>Total: KSh {(order.subtotal_cents / 100).toLocaleString()}</p>
        <p>Status: <strong>{order.status}</strong></p>
        <p>Payment: <strong>{order.payment_status}</strong></p>
      </div>
    </div>
  );
}
```

Now the WhatsApp confirmation can include a link: `https://mctaba.co.ke/track/ABCD1234`. Customers click, see their order status, no login needed.

The short id prefix matching is not bulletproof -- if two orders happen to have the same first 8 characters of UUID, the first match wins. The collision probability is low (1 in 4 billion per pair). For a shop that does a million orders you would use a proper short-code field; for Project 3 this is fine.

---

## Checkpoint

1. Placing a real order and paying it on sandbox triggers two WhatsApp sends: one to the customer, one to the owner.
2. Admin dashboard plays a chime when a new order lands (requires prior click).
3. A cancelled payment shows a retry button that fires a fresh STK Push.
4. Rate-limited retries: clicking retry twice within 30 seconds returns an error.
5. Admin moving an order `paid -> shipped` sends a "your order has been dispatched" WhatsApp to the customer.
6. A COD order can be marked `pending -> delivered` directly, simultaneously setting `payment_status = success`.
7. Empty admin orders page shows a helpful message, not a bare empty table.
8. `/track/ABCD1234` works and shows the order.

Commit:

```bash
git add .
git commit -m "feat: lifecycle notifications retry cod and track page"
```

---

## What You Learned

- Two-channel notifications: customer and owner.
- Fire-and-forget everywhere, with `console.error` on failure.
- Retries need server-side rate limiting, not just UI debouncing.
- COD needs its own lifecycle shortcut; do not try to force it through M-Pesa.
- Empty states matter -- treat them as UX, not afterthoughts.
- Public tracking pages let unauthenticated users check their own orders without logins.

Tomorrow is the recap, and the weekend ships **Project 3** -- E-commerce Lite with WhatsApp Updates, the first shippable roadmap project. You are ready.
