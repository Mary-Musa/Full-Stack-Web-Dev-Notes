# Week 18 - Day 2 Assignment

## Title
Idempotency Deep Dive -- Preventing Double Charges

## Overview
Yesterday you blocked fake webhooks. Today you make sure REAL webhooks that arrive twice (for example, because Daraja retried) do not result in double-charging. You will implement idempotency keys for every mutating endpoint and verify each one catches duplicates.

## Learning Objectives Assessed
- Use a unique constraint as an idempotency guard
- Store a hashed request id for cross-provider idempotency
- Write tests that prove duplicate requests are safe
- Explain the difference between at-least-once and exactly-once

## Prerequisites
- Day 1 completed

## AI Usage Rules

**Ratio:** 45/55. **Habit:** Audit every security line. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Test fixtures and helpers.
- **NOT ALLOWED FOR:** The idempotency check logic itself.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Idempotency table

**What to do:**
```sql
CREATE TABLE idempotency_keys (
  key TEXT PRIMARY KEY,
  response_body TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_idempotency_keys_created ON idempotency_keys(created_at);
```

Load it.

**Expected output:**
Table exists.

### Task 2: Idempotency helper

**What to do:**
Create `lib/idempotency.js`:

```javascript
import pool from "./db";

export async function withIdempotency(key, handler) {
  if (!key) throw new Error("Missing idempotency key");

  // Try to insert -- if the key exists, this will fail
  const { rows } = await pool.query(
    "INSERT INTO idempotency_keys (key) VALUES ($1) ON CONFLICT (key) DO NOTHING RETURNING key",
    [key]
  );

  if (rows.length === 0) {
    // Duplicate -- return cached response
    const { rows: existing } = await pool.query(
      "SELECT response_body FROM idempotency_keys WHERE key = $1",
      [key]
    );
    return JSON.parse(existing[0].response_body || "null");
  }

  // First request -- run the handler and cache the response
  const result = await handler();
  await pool.query(
    "UPDATE idempotency_keys SET response_body = $1 WHERE key = $2",
    [JSON.stringify(result), key]
  );
  return result;
}
```

Hand-typed. No AI.

**Expected output:**
Helper works.

### Task 3: Use it in the M-Pesa callback

**What to do:**
Wrap the callback handler. The idempotency key is `CheckoutRequestID`:

```javascript
const result = await withIdempotency(
  `mpesa:${checkoutId}`,
  async () => {
    // your existing callback logic
    return { orderId: order.id, status: "paid" };
  }
);
```

Now if Daraja retries the callback, the second call returns the cached result instead of re-processing.

**Expected output:**
Replayed callbacks return the same response without side effects.

### Task 4: Stripe and Airtel

**What to do:**
Do the same for Stripe (`stripe:${event.id}`) and Airtel (`airtel:${reference}`).

**Expected output:**
All three providers are idempotent.

### Task 5: Integration test

**What to do:**
Write a test `tests/idempotency.test.js` that:
1. Posts a fake callback once -- asserts status is updated.
2. Posts the exact same callback again -- asserts status is unchanged and response is identical.
3. Posts a different callback -- asserts it is processed normally.

**Expected output:**
All three assertions pass.

## Stretch Goals (Optional - Extra Credit)

- Add a cron job that deletes idempotency keys older than 30 days.
- Use a hash of the request body instead of a provider-specific id.
- Return a `425 Too Early` when the first request is still in flight (race condition).

## Submission Requirements

- **What to submit:** Repo, helper, tests, `AI_AUDIT.md`.
- **Deadline:** End of Day 2

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Idempotency table + unique constraint | 15 | Schema correct. |
| withIdempotency helper | 30 | Hand-typed. Uses INSERT ... ON CONFLICT. |
| M-Pesa callback wrapped | 15 | Duplicate callbacks return cached. |
| Stripe and Airtel wrapped | 20 | All three providers safe. |
| Integration test | 15 | Three assertions pass. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Two round trips for idempotency.** The INSERT ... ON CONFLICT approach does it in one query -- faster and race-safe.
- **Using `SELECT WHERE` before `INSERT`.** Classic race condition -- two simultaneous requests both see "no existing key" and both insert.
- **Caching responses forever.** Add a cleanup cron for old keys.

## Resources

- Day 2 reading: [Idempotency Deep Dive.md](./Idempotency%20Deep%20Dive.md)
- Week 18 AI boundaries: [../ai.md](../ai.md)
