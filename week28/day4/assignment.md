# Week 28 - Day 4 Assignment

## Title
Wallets And Isolation Testing

## Overview
Day 4 adds a wallet table per tenant (balance in cents), credit/debit helpers in a transaction, and a full cross-tenant leak test suite so you can PROVE isolation.

## Learning Objectives Assessed
- Model an append-only ledger for a wallet balance
- Credit and debit in a transaction
- Test cross-tenant leakage systematically
- Keep money in integer cents only

## Prerequisites
- Days 1-3 completed

## AI Usage Rules

**Ratio this week:** 40% manual / 60% AI
**Habit:** Capstone build. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Test scaffolding.
- **NOT ALLOWED FOR:** Ledger logic or tenant isolation proofs.
- **AUDIT REQUIRED:** Yes. Money audit AND isolation audit required.

## Tasks

### Task 1: Ledger table

**What to do:**
```sql
CREATE TABLE wallet_entries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  amount_cents BIGINT NOT NULL,  -- positive = credit, negative = debit
  reason TEXT NOT NULL,
  reference TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
ALTER TABLE wallet_entries ENABLE ROW LEVEL SECURITY;
CREATE POLICY wallet_tenant_isolation ON wallet_entries
  USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

Balance = `SUM(amount_cents)`. No separate balance column.

**Expected output:**
Migration runs.

### Task 2: Credit/debit helpers

**What to do:**
```javascript
async function credit(db, amountCents, reason, reference) {
  if (!Number.isInteger(amountCents) || amountCents <= 0) throw new Error("invalid");
  await db.query(
    "INSERT INTO wallet_entries (tenant_id, amount_cents, reason, reference) VALUES (current_setting('app.tenant_id')::uuid, $1, $2, $3)",
    [amountCents, reason, reference]
  );
}
async function debit(db, amountCents, reason, reference) { ... }
async function balance(db) {
  const { rows } = await db.query("SELECT COALESCE(SUM(amount_cents), 0) AS balance FROM wallet_entries");
  return Number(rows[0].balance);
}
```

**Expected output:**
Helpers work.

### Task 3: Isolation test suite

**What to do:**
Write tests that for each tenant-owned table attempt to:
- Fetch row by ID belonging to another tenant (expect 404)
- Update row by ID belonging to another tenant (expect 404)
- Delete row by ID belonging to another tenant (expect 404)

Run for every tenant-owned table. Name the file `tests/isolation.test.js`.

**Expected output:**
All tests pass.

### Task 4: Reference smoke

**What to do:**
In `isolation-proof.md`, document:
- Your proof that RLS is active on every tenant-owned table
- The query you run to verify (`SELECT tablename FROM pg_tables WHERE schemaname = 'public';` then `\d table` for each)
- The result of running the isolation test suite

**Expected output:**
Committed.

### Task 5: Money audit

**What to do:**
`AI_AUDIT.md` Money Audit section lists every line that touches wallet_entries. For each: `self` or `AI`. Re-read each line.

**Expected output:**
Audit committed.

## Stretch Goals (Optional - Extra Credit)

- Nightly reconciliation job that compares SUM to expected.
- Minimum-balance check on debit.
- Rolling 30-day balance snapshot for speed.

## Submission Requirements

- **What to submit:** Repo, ledger, helpers, isolation tests, audits, `AI_AUDIT.md`.
- **Deadline:** End of Day 4

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Ledger table | 15 | RLS on. |
| Credit/debit helpers | 20 | Integer cents only. |
| Isolation test suite | 30 | Every tenant-owned table covered. |
| Isolation proof | 15 | Documented. |
| Money audit | 15 | Every line classified. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Balance as a column.** Prefer a ledger.
- **Float amounts anywhere.** Cents integers only.
- **Partial test coverage.** Every tenant-owned table gets the same three checks.

## Resources

- Day 4 reading: [Wallets and Isolation Testing.md](./Wallets%20and%20Isolation%20Testing.md)
- Week 28 AI boundaries: [../ai.md](../ai.md)
