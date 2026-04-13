# Week 17 - AI Boundaries

**Ratio this week: 30% Manual / 70% AI**
**Habit introduced: "Provider docs first, implementation second."**
**Shift from last week: Same ratio, new habit for provider integration.**

You add Stripe and Airtel to the shop, then build a unified payments interface that treats all three providers (M-Pesa, Stripe, Airtel) as interchangeable behind one contract. This is classic multi-provider abstraction work -- common in real fintech.

The money rule still holds. Provider-specific quirks are a new category: each provider lies in different ways about different things. The habit is to read each provider's real docs, in order, before writing any integration.

---

## What You MUST Do Manually (30%)

### Day 1 -- Stripe foundations
- Create a Stripe test account yourself. Read the Stripe API overview and Payment Intents docs.
- Write your first Stripe PaymentIntent creation by hand. Use the Stripe Node SDK.
- Handle the webhook for `payment_intent.succeeded` manually. Verify the webhook signature using Stripe's library.
- **Money rule:** every amount, currency, idempotency key is manual.

### Day 2 -- Airtel Money integration
- Read Airtel Money's API docs. They are thinner than Stripe's and AI will try to fill in gaps -- do not let it.
- Build the OAuth flow, the collection request, and the callback handler by hand.
- Airtel has quirks (different auth, polling, response shapes). Document them in your audit.

### Day 3 -- Unified payments interface
- **Design the interface on paper first.** One function signature (`initiatePayment(provider, amount, phone, reference)`) that hides all three providers.
- Write the interface by hand. AI can then implement the adapter for each provider, but you review every line.

### Day 4 -- Provider selection and fallbacks
- Routing logic: if M-Pesa fails, try Airtel; if the customer prefers Stripe (card), go there directly.
- This logic is business-rule-heavy. Manual.

---

## What You CAN Use AI For (70%)

- **Stripe and Airtel SDK wrappers** after you write the first one by hand.
- **Error translation:** mapping provider error codes to user-facing messages.
- **Reconciliation queries** for the admin dashboard.

Forbidden (same as Weeks 10, 16):
- Amount arithmetic.
- Idempotency logic.
- Signature verification.
- Choosing when a payment is "really" successful.

### Good vs bad prompts this week

**Bad:** "Integrate Stripe for me."
**Good:** "Here is my Stripe webhook handler [paste]. I verify the signature and check for idempotency with the event ID. What failure modes am I missing?"

**Bad:** "Build the multi-provider interface."
**Good:** "My interface is `initiatePayment(provider, amount, phone, reference)`. It returns a Promise<PaymentResult>. Is there a field I am likely missing for later reconciliation? I do not want to change the signature after I ship."

---

## Things AI Is Bad At This Week

- **Stripe vs Airtel vs Daraja wire formats.** AI mixes them up. Always specify which provider a question is about.
- **Currency conversion.** If one provider reports in cents and another in shillings, AI will silently mix them. You enforce a single internal unit (always cents) and convert at the edges.
- **Webhook signing schemes.** Different for each provider. Never let AI generate webhook verification -- copy from the official docs.

---

## Core Mental Models For This Week

- **A payment provider is an unreliable narrator.** Every provider claims success slightly differently. Verify with their "get payment" API, not with their callback alone.
- **A unified interface hides differences, not responsibilities.** The adapter handles the quirks; the caller handles the business logic.
- **Reconciliation is the ground truth.** Once a day, compare your DB to each provider's dashboard. Discrepancies are bugs.

---

## This Week's AI Audit Focus

Add a **provider quirks log** with at least three quirks per provider, discovered from reading the docs:

```markdown
## Stripe quirks
1. Payment Intents have idempotency keys, not webhooks.
2. ...

## Airtel quirks
1. ...
```

---

## Assessment

- Live test on all three providers in sandbox.
- Facilitator asks: "what happens when Stripe returns succeeded but your DB has lost the intent ID?" You walk the reconciliation path.
- Defend your unified interface signature in 60 seconds.

---

## One Sentence To Remember

"Every payment provider has quirks. Your job is to know them before you write the integration."
