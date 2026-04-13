# Week 29 - AI Boundaries

**Ratio this week: 25% Manual / 75% AI**
**Habit introduced: "AI is your hands; you are the brain."**
**Shift from last week: The biggest AI ratio of the Capstone Sprint. This week is pure composition: plugging USSD, WhatsApp, and M-Pesa into the tenant-scoped platform you built last week.**

Week 29 is the reward for Week 27's design work and Week 28's careful core. Now you bolt on channels. USSD per tenant, WhatsApp per tenant, payments per tenant, unified inbox and dashboard. Every piece already exists in your codebase. You are wiring, testing, verifying.

The habit is a reminder, not a new rule: you are still the decision-maker, but this week the decisions are few. AI does the typing. You do the verifying.

---

## What You MUST Do Manually (25%)

### Day 1 -- USSD per tenant
- The tenant-lookup from the USSD service code is manual (it is the isolation boundary).
- The state machine itself can be AI-scaffolded if you already have the Week 13 shape.

### Day 2 -- WhatsApp per tenant
- The tenant-lookup from the incoming phone number is manual.
- Everything else is reuse from Week 11 + Week 16.

### Day 3 -- Payments per tenant
- Money rule still applies. Amount validation, idempotency, reconciliation -- manual.
- Integrations are reuse.

### Day 4 -- Unified inbox and dashboard
- The query logic that aggregates across channels is manual.
- The UI is AI-scaffolded.

### Day 5 -- Week 29 demo
- End-to-end test with two tenants. Everything must isolate.

---

## What You CAN Use AI For (75%)

- All UI.
- All scaffolding.
- All documentation.
- All tests (that you review).

Still forbidden:
- Tenant boundary logic.
- Money lines.
- Auth lines.
- Security-relevant code without line-by-line review.

---

## Things AI Is Bad At This Week

- **Cross-tenant regressions.** AI-scaffolded code can silently bypass your tenant filter. Every new feature gets a cross-tenant test.
- **Integration-shaped bugs.** When Week 11's bot meets Week 28's tenant context, AI often misses the wiring. Test carefully.

---

## Core Mental Models For This Week

- **Every feature is "feature with tenant scope".** There is no "feature without tenant scope".
- **Composition is the art of Week 29.** You are not inventing; you are connecting.
- **Tests are the proof of composition.** Every new connection gets a test.

---

## This Week's AI Audit Focus

List every feature you shipped this week and whether it has a cross-tenant test. Missing tests = failed audit.

---

## Assessment

- Facilitator plays a tenant. Sends a WhatsApp. Tests the USSD code. Pays. Refunds. All flows.
- Facilitator plays a second tenant. Same flows. Verifies no cross-tenant bleed.

---

## One Sentence To Remember

"Composition is where multi-tenant apps break. Test every join between a feature and the tenant boundary."
