# Week 18 Weekend: Payment Hardening Pass

No new features. Apply every defence you built this week uniformly across the Project 3 shop. By Monday morning your shop survives every hostile scenario on the audit list below.

**Estimated time:** 4-6 hours.

**Deadline:** Monday morning before Week 19.

---

## The Audit Checklist

Work through this list in order. Each item is binary: done or not done.

### Webhook security
- [ ] Stripe webhook verifies signature against raw body.
- [ ] M-Pesa callback is IP allow-listed against Safaricom's published ranges.
- [ ] M-Pesa callback URL has a secret token segment (not publicly guessable).
- [ ] Airtel callback verifies HMAC with timing-safe compare.
- [ ] All webhook endpoints are rate-limited to 60/minute/IP.
- [ ] All webhook bodies are logged to `webhook_log` (without PII).

### Idempotency
- [ ] Every state-changing webhook handler uses `UPDATE ... WHERE not-already` atomic transition.
- [ ] M-Pesa handler uses `event.id` or an equivalent dedup key.
- [ ] Stripe handler uses `webhook_events` dedup table.
- [ ] Successful state changes write to the `outbox` table in the same transaction.
- [ ] A worker processes the outbox every 5 seconds.
- [ ] Outgoing Stripe calls use `idempotencyKey`.

### Refunds
- [ ] `refunds` table exists with `status` state machine.
- [ ] `createRefund` bounds by total refundable.
- [ ] Each provider's refund function handles the async/sync callback pattern.
- [ ] Admin UI has a refund form per order with amount and reason.
- [ ] Refund success fires a WhatsApp to the customer ("Your refund of KSh X has been processed").

### Reconciliation
- [ ] `reconcile.worker.js` runs at 2am daily.
- [ ] Mismatches are written to `reconciliation_mismatches`.
- [ ] `/admin/reconciliation` shows open mismatches.
- [ ] Resolve button for "provider says paid, db says not paid" auto-transitions and fires WhatsApp.

### Other
- [ ] No secrets in git (run `git grep "sk_test" && git grep "AIRTEL_CLIENT"` to verify).
- [ ] No `console.log` of PII (phone numbers, names, emails) in production code.
- [ ] All inbound requests with invalid signatures return 401, not 500.
- [ ] All errors thrown in payment services are caught at the route boundary and returned as clean JSON.

---

## Hostile Scenario Tests

Run each of these and prove your shop survives.

### Test 1: Fake Stripe callback
POST a fake `checkout.session.completed` event with a wrong signature to `/api/webhooks/stripe`. Expected: 400, no database write.

### Test 2: Fake M-Pesa callback from local
POST a real-shape M-Pesa callback from your laptop (not from Safaricom's IPs). Expected: 403.

### Test 3: Duplicate callback storm
POST 50 copies of the same real Stripe callback in parallel. Expected: order transitions exactly once; outbox has exactly one row.

### Test 4: Out-of-order callbacks
POST a `shipped` event before a `paid` event. Expected: both are persisted; final status is either `paid` or `shipped` depending on your resolution policy; neither fails.

### Test 5: Refund of unpaid order
Try to refund an order that has not been paid. Expected: clean error, no refund row created.

### Test 6: Over-refund
Refund 60% of an order, then 50% more. Expected: second refund rejected.

### Test 7: Rate limit
Fire 100 webhook requests in one minute from the same IP. Expected: after 60, you get 429.

### Test 8: Reconciliation missed callback
Mark an order as `initiated` in the database but the provider sandbox has it as succeeded. Run reconcile. Expected: mismatch surfaces on `/admin/reconciliation`.

---

## Deliverables

- [ ] All audit checklist items checked.
- [ ] All hostile scenario tests pass with screenshots/logs captured.
- [ ] A `SECURITY.md` file in the repo root listing what you protected and how.
- [ ] A `RUNBOOK.md` file with "if X happens, do Y" for the top 5 likely incidents.

---

## Grading Rubric (100 pts)

| Area | Points |
|---|---|
| All audit items done | 40 |
| All hostile tests pass | 30 |
| SECURITY.md quality | 10 |
| RUNBOOK.md quality | 10 |
| No regressions (Project 3 still works end to end) | 10 |

---

## Hints

**Do not add features.** This week's grade is about discipline, not scope. Resist the urge to build a shiny new admin page. Every hour you spend on new features is an hour not spent verifying the old ones.

**Pair on the hostile tests.** Tests 1-3 are worth doing with a partner who is trying to break your shop. Sit next to each other, each with a shell, and take turns attacking and defending. This is how real security reviews work.

**Screenshots, everywhere.** Every passing test should have a screenshot of the attack attempt and the defence. These go in SECURITY.md as evidence. Do not trust "I think I tested it".

**Runbook short is better than long.** Five scenarios, one page total. Examples: "Customer says they paid but order is pending" -> "Check webhook_log for the reference; if Safaricom shows success, click Resolve on /admin/reconciliation".

Good luck. Next week is the Payments Service module extraction.
