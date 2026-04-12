# Week 18, Day 4: Reconciliation

By the end of today, your shop has a daily reconciliation job that compares what each payment provider says happened yesterday with what your database says happened. Mismatches show up on an admin page. The day starts with honesty: "money in, money out, missing, extra."

**Prior-week concepts you will use today:**
- Provider status query APIs (Week 16-17)
- The unified payments interface (Week 17, Day 3)
- SQL aggregates and joins (Week 12)

**Estimated time:** 3 hours

---

## Why Reconcile

Webhooks are best-effort. Networks drop. Providers have bugs. Your server crashes. Out of 1000 transactions, you will lose 1 to 10 callbacks in a year. Without reconciliation, that is 1-10 orders that silently stay `pending` while the customer has the goods and the money is in your account.

Reconciliation is the nightly truth-check: ask each provider for its record of the day, compare to your database, surface the differences.

Three classes of mismatch:

1. **Provider says paid, database says not paid.** A missed callback. Fix: mark the order paid, trigger the notification retroactively.
2. **Database says paid, provider says not paid.** A fraudulent or buggy marking. Very rare if your webhook verification is tight. Fix: investigate, roll back the order.
3. **Amounts disagree.** Either the customer paid a different amount than you expected (weird) or your database has the wrong amount (worse). Fix: investigate manually.

A good reconciliation job does not try to auto-fix everything. It surfaces the list and lets a human decide.

---

## The M-Pesa Side: Daily Transaction Query

Daraja has an API for pulling transaction history for your shortcode by date. Example (simplified):

```javascript
async function fetchMpesaTransactionsForDate(date) {
  const token = await getDarajaToken();
  // Note: Daraja does not have a single "list transactions by date" endpoint.
  // You either poll for each known CheckoutRequestID individually, or
  // fetch the "B2C Account Balance" + transaction statement.
  // On the sandbox you are limited; on production you can download the
  // transaction CSV from the Safaricom portal daily and import it.

  // Programmatic approach: for each known CheckoutRequestID from yesterday,
  // call the STK Push Query API.
  const { rows } = await query(
    `SELECT id, mpesa_checkout_request_id, payment_status
     FROM orders
     WHERE created_at::date = $1
       AND payment_method = 'mpesa'
       AND mpesa_checkout_request_id IS NOT NULL`,
    [date]
  );

  const results = [];
  for (const order of rows) {
    try {
      const status = await queryDarajaStatus(order.id);
      results.push({
        orderId: order.id,
        providerStatus: status.paymentStatus,
        dbStatus: order.payment_status,
      });
    } catch (err) {
      console.error(`Failed to query Daraja for ${order.id}:`, err);
    }
  }

  return results;
}
```

This is a "pull-by-id" reconciliation rather than a true daily report. It works because you already know every order id from yesterday -- just ask Daraja about each one.

For production, supplement with the Safaricom MPesa portal's daily statement CSV and cross-check. The CSV has a reference column you can match on `mpesa_checkout_request_id` (or the shorter `MpesaReceiptNumber` if you captured it on successful payments).

---

## Airtel and Stripe

**Stripe** has a proper history API:

```javascript
async function fetchStripeChargesForDate(dateString) {
  const dayStart = Math.floor(new Date(dateString + "T00:00:00Z").getTime() / 1000);
  const dayEnd = dayStart + 86400;

  const charges = [];
  let hasMore = true;
  let startingAfter = undefined;

  while (hasMore) {
    const res = await stripe.charges.list({
      created: { gte: dayStart, lt: dayEnd },
      limit: 100,
      starting_after: startingAfter,
    });
    charges.push(...res.data);
    hasMore = res.has_more;
    if (hasMore) startingAfter = res.data[res.data.length - 1].id;
  }

  return charges;
}
```

You get an authoritative list of every charge Stripe recorded for your account that day. Match by session id or payment intent id against your database.

**Airtel** has a similar "transaction listing" endpoint, though it tends to require paginated polling like the above.

---

## The Reconciliation Runner

Put it all in one function:

```javascript
// server/services/reconciliation.service.js
const payments = require("./payments");
const { query } = require("../config/db");

async function reconcileDate(date) {
  // 1. Pull provider views
  const mpesaProviderData = await fetchMpesaTransactionsForDate(date);
  const stripeProviderData = await fetchStripeChargesForDate(date);
  // airtel similar

  // 2. Pull DB view of the day
  const { rows: dbOrders } = await query(
    `SELECT id, payment_method, payment_status, subtotal_cents,
            mpesa_checkout_request_id, stripe_session_id, airtel_reference
     FROM orders
     WHERE created_at::date = $1
       AND is_test = FALSE`,
    [date]
  );

  // 3. Compare
  const mismatches = [];

  for (const order of dbOrders) {
    if (order.payment_method === "mpesa") {
      const provider = mpesaProviderData.find((p) => p.orderId === order.id);
      if (!provider) continue;
      if (provider.providerStatus !== order.payment_status) {
        mismatches.push({
          orderId: order.id,
          method: "mpesa",
          dbStatus: order.payment_status,
          providerStatus: provider.providerStatus,
        });
      }
    }
    if (order.payment_method === "stripe") {
      const charge = stripeProviderData.find((c) => c.metadata?.order_id === order.id);
      if (!charge) continue;
      const providerStatus = charge.status === "succeeded" ? "success" : "failed";
      if (providerStatus !== order.payment_status) {
        mismatches.push({
          orderId: order.id,
          method: "stripe",
          dbStatus: order.payment_status,
          providerStatus,
        });
      }
    }
    // airtel similar
  }

  // 4. Write to a reconciliation_runs table for audit
  const { rows: runRows } = await query(
    `INSERT INTO reconciliation_runs (run_date, mismatch_count, completed_at)
     VALUES ($1, $2, NOW())
     RETURNING id`,
    [date, mismatches.length]
  );
  const runId = runRows[0].id;

  for (const m of mismatches) {
    await query(
      `INSERT INTO reconciliation_mismatches (run_id, order_id, method, db_status, provider_status)
       VALUES ($1, $2, $3, $4, $5)`,
      [runId, m.orderId, m.method, m.dbStatus, m.providerStatus]
    );
  }

  return { runId, mismatchCount: mismatches.length };
}

module.exports = { reconcileDate };
```

Schema:

```sql
CREATE TABLE reconciliation_runs (
  id BIGSERIAL PRIMARY KEY,
  run_date DATE NOT NULL,
  mismatch_count INTEGER NOT NULL,
  started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at TIMESTAMPTZ
);

CREATE TABLE reconciliation_mismatches (
  id BIGSERIAL PRIMARY KEY,
  run_id BIGINT NOT NULL REFERENCES reconciliation_runs(id),
  order_id UUID NOT NULL,
  method TEXT NOT NULL,
  db_status TEXT,
  provider_status TEXT,
  resolved_at TIMESTAMPTZ,
  resolution_note TEXT
);
```

---

## Scheduling The Run

For the Marathon, use `node-cron` to run nightly at 2 a.m. (when traffic is low and provider APIs are less busy):

```javascript
// server/workers/reconcile.worker.js
const cron = require("node-cron");
const recon = require("../services/reconciliation.service");

cron.schedule("0 2 * * *", async () => {
  const yesterday = new Date(Date.now() - 86400000).toISOString().slice(0, 10);
  console.log("Running reconciliation for", yesterday);
  try {
    const result = await recon.reconcileDate(yesterday);
    console.log(`Reconciliation complete: ${result.mismatchCount} mismatches`);
  } catch (err) {
    console.error("Reconciliation failed:", err);
  }
});
```

Wire the worker into `server/index.js` with `require("./workers/reconcile.worker")` so it starts on boot.

In Week 23 we move this to BullMQ for retry and scale. For now, one cron job is fine.

---

## The Admin Page

```jsx
// app/admin/reconciliation/page.js
import { query } from "@/lib/db";
import { requireAdmin } from "@/app/actions/auth";
import Link from "next/link";

export default async function ReconciliationPage() {
  await requireAdmin();

  const { rows: runs } = await query(
    `SELECT id, run_date, mismatch_count, completed_at
     FROM reconciliation_runs
     ORDER BY run_date DESC
     LIMIT 30`
  );

  const { rows: openMismatches } = await query(
    `SELECT rm.id, rm.order_id, rm.method, rm.db_status, rm.provider_status, r.run_date
     FROM reconciliation_mismatches rm
     JOIN reconciliation_runs r ON r.id = rm.run_id
     WHERE rm.resolved_at IS NULL
     ORDER BY r.run_date DESC`
  );

  return (
    <div className="max-w-4xl mx-auto p-8">
      <h1 className="text-2xl font-bold mb-6">Reconciliation</h1>

      <section className="mb-10">
        <h2 className="font-semibold mb-3">Open mismatches ({openMismatches.length})</h2>
        {openMismatches.length === 0 && <p className="text-green-700">All clear.</p>}
        <ul className="divide-y border rounded">
          {openMismatches.map((m) => (
            <li key={m.id} className="p-3 flex justify-between">
              <div>
                <Link href={`/admin/orders/${m.order_id}`} className="font-mono text-xs">
                  {m.order_id.slice(0, 8)}
                </Link>
                <span className="ml-4">{m.method}</span>
              </div>
              <div>
                <span className="text-sm">db: {m.db_status}</span>
                <span className="mx-2">vs</span>
                <span className="text-sm text-red-600">provider: {m.provider_status}</span>
              </div>
            </li>
          ))}
        </ul>
      </section>

      <section>
        <h2 className="font-semibold mb-3">Recent runs</h2>
        <ul className="text-sm">
          {runs.map((r) => (
            <li key={r.id} className="py-1">
              {r.run_date.toISOString().slice(0, 10)} -- {r.mismatch_count} mismatches
            </li>
          ))}
        </ul>
      </section>
    </div>
  );
}
```

Admins get a single place to see "is everything healthy?". Zero open mismatches is the target state.

---

## Resolving Mismatches

Add a button per mismatch that runs the appropriate fix:

- **Provider says paid, DB says not paid**: call the order's `markPaid` function using the provider's data.
- **Everything else**: mark the mismatch as requiring manual review (`resolution_note`).

```javascript
export async function resolveMismatch(mismatchId, action) {
  await requireAdmin();

  const { rows } = await query(
    "SELECT * FROM reconciliation_mismatches WHERE id = $1",
    [mismatchId]
  );
  const m = rows[0];
  if (!m) return { error: "Mismatch not found" };

  if (action === "mark_paid" && m.provider_status === "success" && m.db_status !== "success") {
    // Rerun the state transition
    await query(
      `UPDATE orders SET payment_status = 'success', status = 'paid', updated_at = NOW()
       WHERE id = $1 AND payment_status != 'success'`,
      [m.order_id]
    );
    // Write to outbox so WhatsApp fires
    await query(
      `INSERT INTO outbox (event_type, payload) VALUES ('order.paid', $1)`,
      [JSON.stringify({ orderId: m.order_id })]
    );
  }

  await query(
    `UPDATE reconciliation_mismatches
     SET resolved_at = NOW(), resolution_note = $1
     WHERE id = $2`,
    [`Resolved via ${action}`, mismatchId]
  );

  return { ok: true };
}
```

---

## Checkpoint

1. Running `await reconcileDate('2026-04-11')` by hand completes without errors and creates a row in `reconciliation_runs`.
2. Simulating a missed callback (mark an order `initiated` in the DB but the provider says it succeeded) surfaces the mismatch.
3. The `/admin/reconciliation` page shows open mismatches.
4. Clicking the resolve button transitions the order to `paid` and writes to the outbox.
5. The worker sends a retroactive WhatsApp confirmation from the outbox.
6. The cron job runs once if you adjust the schedule to fire in the next minute.

Commit:

```bash
git add .
git commit -m "feat: daily reconciliation with mismatch surfacing and resolve actions"
```

---

## What You Learned

- Webhooks are best-effort; reconciliation is how you catch the gaps.
- Pull-by-id reconciliation uses the provider's query API for each known transaction.
- Stripe's charges-list API is cleaner than Daraja's.
- Auto-resolve the easy case (missed callback), surface the hard case (disagreement) for manual review.
- Reconciliation is a daily job, not a real-time check. Accept the lag.

Tomorrow is the recap, and the weekend is payments hardening -- a list of small safety improvements applied across the shop.
