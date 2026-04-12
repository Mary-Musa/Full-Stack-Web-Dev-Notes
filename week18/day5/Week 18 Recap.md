# Week 18, Day 5: Week Recap

This week was all defence -- no new features, just making the existing payment layer correct under pressure. A customer may not notice any difference in normal use, but when things go wrong (and they will), the difference between "hours of panic" and "a clear list of 3 mismatches to resolve" is everything you built this week.

---

## What You Built

1. **Webhook signature verification** (Stripe, Airtel) and IP allow-listing (M-Pesa).
2. **webhook_log table** for dedicated audit of every incoming call.
3. **Rate limiting** on public webhook endpoints.
4. **Atomic check-and-set** state transitions.
5. **Transactional outbox pattern** for reliable side effects.
6. **Event log** for out-of-order callbacks.
7. **Idempotency keys** on outgoing calls.
8. **Refund flows** across M-Pesa B2C, Airtel reversal, Stripe refunds.
9. **Refund state machine** with partial refund support.
10. **Daily reconciliation** comparing provider truth to database state.
11. **Admin reconciliation page** with resolve actions.

---

## Self-Review Questions

**Security**
1. Why must you use the raw body, not the parsed JSON, when verifying a Stripe signature?
2. Why is M-Pesa's webhook security weaker than Stripe's, and what do you do about it?
3. What is `timingSafeEqual` and why do you use it?
4. What is in the `webhook_log` table and what is NOT in it?

**Idempotency**
5. What is the difference between natural idempotency and an idempotency key?
6. Why is `UPDATE ... WHERE payment_status != 'success'` better than "SELECT then UPDATE"?
7. What does the transactional outbox pattern protect you from?
8. Why does out-of-order callback handling benefit from an events log instead of a status column?

**Refunds**
9. Why does M-Pesa need an entirely separate B2C API for refunds?
10. How does your `createRefund` enforce "do not over-refund"?
11. What are the three states of a refund and how do they map to each provider's API?

**Reconciliation**
12. What are the three classes of reconciliation mismatch?
13. Why does reconciliation auto-resolve the "missed callback" case but surface the "amount mismatch" case for manual review?
14. Why do we run reconciliation at 2am?

Target: 12/14 correct.

---

## Peer Coding Session

### Track A: Stress test idempotency

Write a script that fires 100 callbacks in parallel for the same order across several simulated events (payment, shipped, delivered). Assert the final state is correct and the outbox has the right number of events. Make the test part of your test suite.

### Track B: Partial refund chains

Refund 30%, then 20%, then 50% of an order in sequence. Verify the total is 100% and a fourth refund of 10% is rejected. Then test partial refunds with different reasons and make sure the refund list on the admin page groups them sensibly.

### Track C: Reconciliation demo

Plant a deliberate mismatch (set `payment_status = 'initiated'` for an order that Safaricom's sandbox has already completed). Run reconcile. Show the mismatch appearing on the admin page. Click resolve. Show the retroactive WhatsApp firing.

### Track D: Webhook security audit

Try to attack your own webhooks. POST fake callbacks with wrong signatures, wrong IPs, flooded requests. Write up each attack and the defence that stopped it. This is the most educational of the four.

---

## Weekend

The weekend project this week is a **payment hardening pass** -- apply every idempotency, security, and reconciliation check uniformly across your shop. No new features. No new screens. Just making sure nothing you already built can silently drop money.

Next week (Week 19) is the Payments Service module deliverable: extract everything you have built into a clean, documented, reusable package that another shop could drop in. That is the Phase 3 deliverable the syllabus promised.
