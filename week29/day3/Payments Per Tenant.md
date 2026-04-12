# Week 29, Day 3: Payments Per Tenant

By the end of today, customers can pay a tenant via M-Pesa, the payment lands in that tenant's wallet, the order is marked paid, and a confirmation goes out. All of this uses the `mctaba-payments` package from Week 19.

**Estimated time:** 4 hours.

---

## Two Common Patterns For Multi-Tenant Payments

**A. Platform till with split.** All payments go to one Safaricom paybill owned by the platform (you). On callback, you credit the tenant's wallet minus a platform fee. Tenants withdraw via B2C later. This is the simplest but requires KYC/license depending on scale.

**B. Tenant till per tenant.** Each tenant has their own M-Pesa paybill registered in their name. Money goes straight to them; you get a callback for reconciliation. Clean but requires each tenant to get their own Daraja access.

The capstone uses **Pattern A** for simplicity. Platform till, split to tenant wallets. Production SMEs often upgrade to Pattern B.

---

## The Order-To-Payment Flow

1. Customer selects a product via USSD or WhatsApp.
2. Bot confirms the order.
3. API creates an order row (`status=pending`, `tenant_id` set).
4. API calls `mctaba-payments.initiate("mpesa", { orderId, phone, amountCents })`.
5. Customer sees STK prompt, enters PIN.
6. Daraja calls the shared webhook.
7. Webhook looks up the order by `CheckoutRequestID`, finds its tenant_id.
8. Transaction: mark order paid, credit tenant's wallet, decrement stock.
9. Outbox writes an `order.paid` event.
10. Worker picks it up, sends confirmation via the tenant's WhatsApp.

---

## Creating An Order From USSD

Back in the USSD state machine, the `confirm_purchase` state now does the real thing:

```javascript
// api/services/ussd/states/confirm_purchase.js
const { query } = require("../../../config/db");
const payments = require("../../payments");

module.exports = async function confirmPurchase({ input, context, tenant, phoneNumber, client }) {
  if (input !== "1") {
    return { response: "END Cancelled.", nextState: "done", nextContext: {} };
  }

  const productId = context.selectedProductId;
  const { rows: products } = await client.query(
    "SELECT id, price_cents FROM products WHERE id = $1 FOR UPDATE",
    [productId]
  );
  const product = products[0];
  if (!product) return { response: "END Product not found.", nextState: "done", nextContext: {} };

  // Create the customer if new
  const { rows: custRows } = await client.query(
    `INSERT INTO customers (tenant_id, phone) VALUES ($1, $2)
     ON CONFLICT (tenant_id, phone) DO UPDATE SET phone = EXCLUDED.phone
     RETURNING id`,
    [tenant.id, phoneNumber]
  );
  const customerId = custRows[0].id;

  // Create the order
  const { rows: orderRows } = await client.query(
    `INSERT INTO orders (tenant_id, customer_id, total_cents, channel, status)
     VALUES ($1, $2, $3, 'ussd', 'pending')
     RETURNING id`,
    [tenant.id, customerId, product.price_cents]
  );
  const orderId = orderRows[0].id;

  await client.query(
    `INSERT INTO order_items (tenant_id, order_id, product_id, quantity, price_cents_at_purchase)
     VALUES ($1, $2, $3, 1, $4)`,
    [tenant.id, orderId, productId, product.price_cents]
  );

  // Initiate M-Pesa
  try {
    await payments.initiate("mpesa", {
      orderId,
      phone: phoneNumber,
      amountCents: product.price_cents,
    });
  } catch (err) {
    console.error("STK init failed:", err);
    return { response: "END Could not start payment. Try again.", nextState: "done", nextContext: {} };
  }

  return {
    response: `END Check your phone for the M-Pesa prompt. Enter PIN to pay KSh ${(product.price_cents / 100).toLocaleString()}.`,
    nextState: "done",
    nextContext: {},
  };
};
```

The state handler writes the order inside the USSD request's transaction, kicks off STK, and ends the USSD session. The customer enters the PIN on their phone; the callback handles the rest.

---

## The Payment Callback Handler

The `mctaba-payments` package already handles the callback and marks the order paid. You extend it by hooking into the outbox event to credit the wallet:

```javascript
// api/workers/outbox.worker.js
const { query, pool } = require("../config/db");
const walletService = require("../services/wallet.service");
const { notificationsQueue } = require("../queues");

async function processOutboxRow(row) {
  if (row.event_type === "order.paid") {
    const orderId = row.payload.orderId;

    const client = await pool.connect();
    try {
      await client.query("BEGIN");

      const { rows: orderRows } = await client.query(
        "SELECT id, tenant_id, total_cents, customer_id FROM orders WHERE id = $1",
        [orderId]
      );
      const order = orderRows[0];
      if (!order) {
        await client.query("ROLLBACK");
        return;
      }

      await client.query(`SET LOCAL app.current_tenant = $1`, [order.tenant_id]);

      // Credit the tenant's wallet
      await walletService.createWalletTransaction(client, {
        tenantId: order.tenant_id,
        amountCents: order.total_cents,
        kind: "credit",
        ref: `order:${order.id}`,
        note: `Payment for order ${order.id.slice(0, 8)}`,
      });

      // Decrement stock (if not already done)
      await client.query(
        `UPDATE products p
         SET stock = stock - oi.quantity
         FROM order_items oi
         WHERE oi.order_id = $1 AND oi.product_id = p.id`,
        [orderId]
      );

      await client.query("COMMIT");

      // Enqueue the confirmation notification
      const { rows: custRows } = await query(
        "SELECT phone FROM customers WHERE id = $1", [order.customer_id]
      );
      await notificationsQueue.add("whatsappText", {
        tenantId: order.tenant_id,
        to: custRows[0].phone,
        text: `Your order ${order.id.slice(0, 8)} is paid. Thank you!`,
      });
    } catch (err) {
      await client.query("ROLLBACK");
      throw err;
    } finally {
      client.release();
    }
  }
}

module.exports = { processOutboxRow };
```

Everything in one transaction: credit the wallet, deduct the stock, commit. The notification enqueue is after commit because it is fire-and-forget.

---

## Platform Fees

If you want to take a platform fee, split the credit:

```javascript
const platformFee = Math.floor(order.total_cents * 0.02); // 2%
const tenantCredit = order.total_cents - platformFee;

await walletService.createWalletTransaction(client, {
  tenantId: order.tenant_id,
  amountCents: tenantCredit,
  kind: "credit",
  note: `Payment for order ${order.id}`,
});

// A separate platform wallet for fees
await walletService.createWalletTransaction(client, {
  tenantId: PLATFORM_TENANT_ID,
  amountCents: platformFee,
  kind: "fee",
  note: `Platform fee from order ${order.id}`,
});
```

The platform tenant is a special tenant row for fee accounting. Skip this for the capstone unless you want to complicate the demo.

---

## Testing End-To-End

Full walk-through:

1. Sign up Tenant A.
2. Add a product priced at KSh 100.
3. Configure Tenant A's `ussd_code`.
4. Dial the USSD code from sandbox.
5. Navigate to the product, confirm purchase.
6. Enter sandbox M-Pesa PIN.
7. Check the orders page -- order is `paid`.
8. Check the wallet page -- balance is KSh 100 (or KSh 98 with 2% fee).
9. Check the customer's phone -- WhatsApp confirmation arrived.

All in one flow. This is the moment the SME OS goes from "lots of pieces" to "one system".

---

## Checkpoint

1. USSD purchase flow creates an order, triggers STK, and returns a friendly message.
2. Payment callback credits the wallet and decrements stock in one transaction.
3. Customer receives WhatsApp confirmation via the tenant's number.
4. Two tenants handling concurrent orders do not cross-contaminate.
5. The isolation tests from Week 28 still pass.

Commit:

```bash
git add .
git commit -m "feat: multi-tenant payments flow with wallet credits"
```

---

## What You Learned

- Platform tills + wallet splits is the simplest multi-tenant payment pattern.
- The outbox worker hooks into `order.paid` events and does the accounting.
- Notifications are always enqueued after commit, not inside.
- The USSD state handler can run the whole order+payment chain in one transaction.

Tomorrow: the unified dashboard that shows orders from all channels.
