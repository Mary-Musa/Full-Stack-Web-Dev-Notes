# Week 18 - Day 3 Assignment

## Title
Refunds -- Server Actions That Return Money

## Overview
Today you add refunds. The customer wants their money back; you call the provider's refund API, update the order status, and log the refund event. This touches money so everything is hand-written.

## Learning Objectives Assessed
- Call Stripe's refund API
- Call Daraja's B2C reversal API (or simulate -- sandbox limits)
- Call Airtel's refund API
- Surface refund status in the admin dashboard

## Prerequisites
- Days 1-2 completed

## AI Usage Rules

**Ratio:** 45/55. **Habit:** Audit every security line. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Admin UI scaffolding.
- **NOT ALLOWED FOR:** Refund logic. Money rule.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Refunds table

**What to do:**
```sql
CREATE TABLE refunds (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES orders(id),
  provider TEXT NOT NULL,
  provider_reference TEXT,
  amount_cents INTEGER NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'succeeded', 'failed')),
  reason TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Expected output:**
Table created.

### Task 2: Stripe refund Server Action

**What to do:**
Create `app/admin/orders/[id]/refundAction.js`:

```javascript
"use server";
import { stripe } from "@/lib/stripe";
import pool from "@/lib/db";

export async function refundStripeOrder(orderId, reason) {
  const { rows } = await pool.query(
    "SELECT stripe_session_id, total_cents, status FROM orders WHERE id = $1",
    [orderId]
  );
  const order = rows[0];
  if (!order || order.status !== "paid") return { error: "Cannot refund" };

  // Get the Payment Intent id from the session
  const session = await stripe.checkout.sessions.retrieve(order.stripe_session_id);
  const paymentIntentId = session.payment_intent;

  // Issue refund
  const refund = await stripe.refunds.create({
    payment_intent: paymentIntentId,
    reason: reason || "requested_by_customer",
  });

  // Record it
  await pool.query(
    `INSERT INTO refunds (order_id, provider, provider_reference, amount_cents, status, reason)
     VALUES ($1, 'stripe', $2, $3, $4, $5)`,
    [orderId, refund.id, order.total_cents, refund.status, reason]
  );

  // Update order status
  await pool.query(
    "UPDATE orders SET status = 'cancelled' WHERE id = $1",
    [orderId]
  );

  return { success: true, refundId: refund.id };
}
```

Hand-typed. Every money line manual.

**Expected output:**
Refund Server Action works for Stripe test payments.

### Task 3: Admin refund UI

**What to do:**
On your admin order detail page, add a Refund button (only visible if order is `paid` AND `payment_method === "card"`). Clicking it opens a dialog asking for the reason, then calls the Server Action.

**Expected output:**
Admin can refund a test Stripe payment end to end.

### Task 4: M-Pesa and Airtel refunds

**What to do:**
Implement `refundMpesaOrder` and `refundAirtelOrder`. Note:
- Daraja B2C (business-to-customer) is more complex and requires additional credentials. In sandbox, you can log the refund intent without actually pushing money -- document this in your notes.
- Airtel has a reversal API; read the docs before implementing.

For now, these can be stubs that just log the refund and mark the order cancelled. Real B2C integration is a future week's problem.

**Expected output:**
Three refund paths, even if two are stubs.

### Task 5: Refund notes

**What to do:**
In `refund-notes.md`, answer:
- Why is refunding harder than charging?
- What happens if you refund the same payment twice?
- Why do you record the refund in your own DB even when the provider is the source of truth?

Your own words.

**Expected output:**
`refund-notes.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Add a "partial refund" option on Stripe (refund X of Y cents).
- Fetch refund status from Stripe on load to confirm it went through.
- Email/WhatsApp the customer that their refund has started.

## Submission Requirements

- **What to submit:** Repo, refund actions, admin UI, `refund-notes.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 3

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Refunds table | 10 | Schema correct. |
| Stripe refund works end to end | 30 | Test payment refunded. Admin UI. |
| Refunds recorded in DB | 20 | Audit trail in `refunds` table. |
| M-Pesa and Airtel refund stubs | 15 | Admin can trigger even if stubbed. |
| Refund notes | 15 | Three questions answered. |
| Clean commits | 10 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Allowing refunds on unpaid orders.** Guard with `status === 'paid'`.
- **Missing idempotency on refund.** Use the idempotency helper from Day 2.
- **Not updating the order status.** A refunded order is `cancelled`, not `paid`.

## Resources

- Day 3 reading: [Refunds.md](./Refunds.md)
- Week 18 AI boundaries: [../ai.md](../ai.md)
- Stripe refund docs: https://stripe.com/docs/refunds
