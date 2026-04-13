# Week 29 - Day 3 Assignment

## Title
Payments Per Tenant -- M-Pesa Daraja With Shared Code

## Overview
Day 3 brings M-Pesa into the capstone. Per-tenant Daraja credentials (consumer key/secret, shortcode, passkey), shared STK push code, and a callback that credits the right tenant's wallet.

## Learning Objectives Assessed
- Store per-tenant Daraja credentials
- Initiate STK push in the tenant's context
- Verify the callback and credit the wallet in a transaction
- Respect the permanent rule: never trust AI with money

## Prerequisites
- Days 1-2 completed
- Week 10 M-Pesa knowledge

## AI Usage Rules

**Ratio this week:** 25% manual / 75% AI
**Habit:** Integrations week. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Provider boilerplate.
- **NOT ALLOWED FOR:** Ledger updates and callback verification. PERMANENT RULE.
- **AUDIT REQUIRED:** Yes. Money audit mandatory.

## Tasks

### Task 1: daraja_configs table

**What to do:**
```sql
CREATE TABLE daraja_configs (
  tenant_id UUID PRIMARY KEY REFERENCES tenants(id),
  env TEXT NOT NULL CHECK (env IN ('sandbox','production')),
  consumer_key_encrypted TEXT NOT NULL,
  consumer_secret_encrypted TEXT NOT NULL,
  shortcode TEXT NOT NULL,
  passkey_encrypted TEXT NOT NULL,
  callback_url TEXT NOT NULL
);
```

Encrypt all secret fields.

**Expected output:**
Row per tenant.

### Task 2: Initiate STK push

**What to do:**
`POST /v1/orders/:id/pay` with the order's tenant:
- Look up the order amount.
- Look up the tenant's daraja config.
- Get an access token.
- Call STK push with `AccountReference: order.id`.
- Insert a `payment_attempts` row with status `pending`.

Return 202.

**Expected output:**
Sandbox STK push fires.

### Task 3: Callback handler

**What to do:**
`POST /callbacks/mpesa/:tenantId` (or route by shortcode in the payload).

In a transaction:
- Verify the callback shape.
- Find the order by AccountReference.
- Insert a `wallet_entries` credit row for the amount.
- Mark the order as `paid`.
- Mark the payment_attempt as `paid`.

Idempotent: same `mpesa_reference` handled twice is a no-op.

**Expected output:**
End-to-end real sandbox payment updates wallet and order.

### Task 4: Money audit

**What to do:**
`AI_AUDIT.md` Money Audit section lists every file touched:
- STK push call
- Callback handler
- Wallet credit

For each line: `self` or `AI`. Re-read yourself. No exceptions.

**Expected output:**
Audit committed.

### Task 5: Reconciliation notes

**What to do:**
`reconciliation.md`, 5-7 sentences:
- What happens if the callback never arrives?
- How do you handle a duplicate callback?
- How would a nightly reconciliation job work?
- What does "eventual consistency" buy you?

Your own words.

**Expected output:**
Committed.

## Stretch Goals (Optional - Extra Credit)

- Add a refund endpoint.
- Webhook signature verification (if Daraja ever ships it).
- Slack alert on failed callback.

## Submission Requirements

- **What to submit:** Repo, handlers, audit, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 3

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| daraja_configs encrypted | 15 | Secrets not plaintext. |
| STK push per tenant | 25 | Fires correctly. |
| Callback updates wallet in transaction | 30 | End-to-end works, idempotent. |
| Money audit | 20 | Every line re-read. |
| Reconciliation notes | 5 | Four questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Callback without transaction.** Partial updates leak money.
- **Trusting AI with the ledger update.** Permanent rule.
- **Logging PII or amounts to stdout without redaction.** Scrub.

## Resources

- Day 3 reading: [Payments Per Tenant.md](./Payments%20Per%20Tenant.md)
- Week 29 AI boundaries: [../ai.md](../ai.md)
