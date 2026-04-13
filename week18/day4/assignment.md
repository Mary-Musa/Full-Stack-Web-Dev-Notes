# Week 18 - Day 4 Assignment

## Title
Reconciliation -- Daily Check Between Your DB And Your Providers

## Overview
Day 4 is pre-weekend polish. Today you build a reconciliation script that compares your `orders` table against each provider's records, flags discrepancies, and emails you a daily summary. This is how real fintechs catch lost money.

## Learning Objectives Assessed
- Query provider APIs for transactions in a date range
- Compare with local DB to find missing or extra records
- Generate a reconciliation report
- Schedule it to run daily

## Prerequisites
- Days 1-3 completed

## AI Usage Rules

**Ratio:** 45/55. **Habit:** Audit every security line. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Report formatting.
- **NOT ALLOWED FOR:** The diff logic between provider and DB. Money rule.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Reconciliation function

**What to do:**
Create `lib/reconcile.js`:

```javascript
import { stripe } from "./stripe";
import pool from "./db";

export async function reconcileStripe(startDate, endDate) {
  // Pull all Stripe Charges in the date range
  const charges = await stripe.charges.list({
    created: {
      gte: Math.floor(startDate.getTime() / 1000),
      lt: Math.floor(endDate.getTime() / 1000),
    },
    limit: 100,
  });

  // Pull local orders with Stripe payments in the same range
  const { rows: localOrders } = await pool.query(
    `SELECT id, stripe_session_id, total_cents, status FROM orders
     WHERE created_at >= $1 AND created_at < $2 AND payment_method = 'card'`,
    [startDate, endDate]
  );

  const report = { matched: 0, extraInStripe: [], extraInDb: [], amountMismatches: [] };

  for (const charge of charges.data) {
    const metadata = charge.metadata;
    const orderId = metadata?.order_id;
    if (!orderId) {
      report.extraInStripe.push(charge.id);
      continue;
    }
    const localOrder = localOrders.find((o) => o.id === orderId);
    if (!localOrder) {
      report.extraInStripe.push(charge.id);
    } else if (charge.amount !== localOrder.total_cents) {
      report.amountMismatches.push({ chargeId: charge.id, stripe: charge.amount, local: localOrder.total_cents });
    } else {
      report.matched++;
    }
  }

  return report;
}
```

Hand-typed. Amount comparison is the core.

**Expected output:**
Function returns a report object.

### Task 2: Reconciliation endpoint

**What to do:**
Create `app/api/admin/reconcile/route.js`:

```javascript
import { reconcileStripe } from "@/lib/reconcile";
import { NextResponse } from "next/server";

export async function GET(req) {
  // Auth: admin only
  const startDate = new Date(req.nextUrl.searchParams.get("start") || Date.now() - 86400000);
  const endDate = new Date(req.nextUrl.searchParams.get("end") || Date.now());
  const report = await reconcileStripe(startDate, endDate);
  return NextResponse.json(report);
}
```

**Expected output:**
Endpoint returns a JSON report.

### Task 3: Display in admin

**What to do:**
Create `app/admin/reconcile/page.js` (Server Component). Fetch the report for yesterday and render:

- How many matched
- Which records are in Stripe but not your DB
- Which are in your DB but not Stripe
- Which have amount mismatches

Style with Tailwind. Bold red highlights for any non-zero mismatch.

**Expected output:**
Screenshot `day4-reconcile.png`.

### Task 4: Pre-weekend checklist

**What to do:**
Create `CHECKLIST.md`:

```markdown
## Week 18 Day 4 Pre-Weekend Checklist

- [ ] All webhooks verify signatures
- [ ] Idempotency table preventing double processing
- [ ] Stripe refund via admin UI
- [ ] M-Pesa and Airtel refund stubs
- [ ] Refunds recorded in refunds table
- [ ] Reconciliation script compares Stripe to local
- [ ] Admin reconcile page shows discrepancies
- [ ] Money audit is current (zero AI in money code)
- [ ] Repo pushed and clean
```

Tick honestly.

**Expected output:**
Checklist committed.

### Task 5: Disaster scenario

**What to do:**
In `disaster-scenarios.md`, describe what happens in each of these situations and what your code does about it (5-8 sentences each):

1. Daraja's callback never arrives (network drop)
2. Stripe webhook arrives 30 minutes late
3. A refund succeeds on Stripe but your DB update fails
4. Two customers race to pay for the last item in stock

Your own words.

**Expected output:**
`disaster-scenarios.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Extend reconciliation to M-Pesa via Daraja's transaction query API.
- Schedule the reconcile endpoint to run daily via GitHub Actions cron.
- Email the report as a daily digest.

## Submission Requirements

- **What to submit:** Repo, reconcile files, `CHECKLIST.md`, `disaster-scenarios.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 4

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| reconcileStripe function | 25 | Produces a real report. |
| /api/admin/reconcile endpoint | 15 | Protected by admin auth. |
| Admin reconcile UI | 20 | Discrepancies highlighted. |
| CHECKLIST complete | 10 | Honest. |
| Disaster scenarios notes | 20 | Four scenarios answered thoughtfully. |
| Clean commits | 10 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Comparing ints loosely.** `charge.amount == order.total_cents` works only if both are integers. Use strict `===` and cast carefully.
- **Paginating wrong.** Stripe's list is limited to 100. Paginate if you expect more.
- **Running reconcile against production with no rate limit.** It can pull thousands of records.

## Resources

- Day 4 reading: [Reconciliation.md](./Reconciliation.md)
- Week 18 AI boundaries: [../ai.md](../ai.md)
