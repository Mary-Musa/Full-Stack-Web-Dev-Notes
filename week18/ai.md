# Week 18 - AI Boundaries

**Ratio this week: 45% Manual / 55% AI** (SECURITY DIP)
**Habit introduced: "Audit every security line AI writes."**
**Shift from last week: AI room bumps back down. This is a deliberate security dip -- you are going deep on webhook security, idempotency, and refunds.**

The ratio has been loosening steadily from Week 10 onwards. This week it snaps back. Week 18 is where real production-grade payment hardening happens: HMAC signature verification on every callback, bulletproof idempotency (the outbox pattern), refund flows across three providers, and daily reconciliation jobs.

The Week 10 and Week 12 rules (never trust AI with money, never trust AI with auth) are joined this week by a third: **never trust AI with security-critical code without a full line-by-line audit**. AI can scaffold. AI cannot ship.

---

## Why The Ratio Dipped

You have been at 30% manual for two weeks, riding the ability to compose known patterns. Security is different. Every vulnerability class (signature bypass, replay attack, race condition, refund duplication) has the same shape: "the happy path works, but the attacker's path does something unexpected". AI writes happy paths. You write attacker paths.

Bump to 45% this week is not a punishment; it is recalibration. Next week (Week 19) you are back at 35/65 for the payments package extraction, which is scaffolding-heavy.

---

## What You MUST Do Manually (45%)

### Day 1 -- Webhook security
- HMAC-SHA256 verification from scratch (you did this for WhatsApp in Week 11; this week you do it again for Stripe and extend to all providers).
- Timing-safe comparison: `crypto.timingSafeEqual` -- know why it matters vs `===`.
- Replay protection: timestamp windows + nonce tracking.
- Every webhook you already have gets audited this week. Missing verification anywhere is a fail.

### Day 2 -- Idempotency deep dive
- The outbox pattern: on every state change, write to an `outbox` table in the same transaction. A background worker publishes from the outbox.
- Write the outbox pattern by hand for your order state machine.
- Deduplication keys: a unique constraint on `(provider, event_id)` in your events table so duplicate callbacks cannot double-process.
- Test with a deliberately-replayed webhook. Your code must no-op on the second call.

### Day 3 -- Refunds
- Full refund flow for Stripe, M-Pesa (B2C), and Airtel. Each is different.
- Partial refunds (where supported).
- Refund race conditions: what happens if a refund and a new charge for the same order interleave? Draw it on paper first.

### Day 4 -- Reconciliation
- Daily cron job that pulls the last 24 hours of transactions from each provider and compares them to your DB.
- Any discrepancies go into a `reconciliation_mismatches` table and notify an admin.
- This job is manual because the query and comparison logic are the ground truth of your financial health.

---

## What You CAN Use AI For (55%)

- **Explaining HMAC** after you have written the code.
- **Generating test fixtures** (fake webhook payloads with correct and incorrect signatures).
- **Drafting the reconciliation dashboard UI.**
- **Refactoring your outbox worker** after it works.

Forbidden:
- Writing signature verification (rule #3 this week).
- Choosing the idempotency strategy.
- Writing the refund flow (money rule).
- Writing reconciliation logic (financial ground truth).

### Good vs bad prompts this week

**Bad:** "Add webhook security to my app."
**Good:** "Here is my Stripe webhook verification [paste]. I use `stripe.webhooks.constructEvent` with the signing secret. What attack classes does this not protect me from?"

**Bad:** "Write my refund handler."
**Good:** "I am designing a refund flow for M-Pesa B2C. The provider is async: the refund request returns an ID and I get a callback later. What state machine should my refund row have?"

---

## Things AI Is Bad At This Week

- **Constant-time comparisons.** AI sometimes writes `hash1 === hash2`, which leaks timing info. Always use the crypto library's timing-safe helper.
- **Replay protection.** AI often forgets about it. "Signature is valid" and "this is not a replay of an earlier valid signature" are two different checks.
- **Refund idempotency.** AI will happily write a refund handler that can double-refund under load. Always use a unique key from the provider.
- **Reconciliation cursoring.** AI rarely handles pagination over provider APIs correctly. Test your cursor logic manually.

---

## Core Mental Models For This Week

- **Security is the set of invariants an attacker cannot violate.** "The amount charged equals the amount displayed" is an invariant. Your job is to enforce it at every layer.
- **Idempotency is "doing the same operation twice has the same effect as once".** Write your mutations so this is true.
- **Reconciliation is truth-checking.** Trust your DB only as far as it agrees with the provider's records.
- **Audit logs are forever.** Every state change writes a row. Never delete.

---

## This Week's AI Audit Focus

Add a required section: **Security audit per file.** For every file touched this week, list:
- Every security-relevant line (signature check, idempotency key, amount verification)
- Who wrote it
- When it was last manually reviewed
- Any AI-generated line gets a line-by-line annotation explaining what it does

---

## Assessment

- Facilitator throws three attacks at your endpoints: replayed webhook, tampered signature, wrong amount. Your code must reject all three.
- Walk through the outbox pattern on a whiteboard.
- Live test: run your reconciliation job against a deliberately-mismatched DB row. Your job should find and report it.

### Live Rebuild Check

Facilitator deletes your signature verification. You rewrite it from memory. If you cannot, this is a serious gap -- come back with proof.

---

## One Sentence To Remember

"The attacker's path is the only path that matters. AI writes happy paths. You write attacker paths."
