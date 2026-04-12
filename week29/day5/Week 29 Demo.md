# Week 29, Day 5: Full Demo and Week 29 Close

By the end of today, the SME OS is ready for a full demo: a customer dials a USSD code, buys a product, receives a WhatsApp confirmation, and the tenant sees everything in their dashboard in real time. This is what Week 27's spec described, and it works end to end.

**Estimated time:** 3-4 hours (testing, polish, screenshots).

---

## The Demo Script

Rehearse this until it is smooth. Three tenants, three customers, five minutes total.

1. **Setup** (pre-rehearsed):
   - Tenant A (Kilimani Kitchen) has 3 products and USSD code `*384*1234*1#`.
   - Tenant B (Westlands Gym) has 2 services and USSD code `*384*1234*2#`.
   - Both tenants' dashboards are open in two browser tabs.

2. **Customer Story** (on screen, narrated):
   - A customer named Brian dials `*384*1234*1#`.
   - Menu appears: "Kilimani Kitchen. 1. Browse products. 2. Contact us."
   - Brian picks 1. Sees three products.
   - Picks "Ugali and Nyama -- KSh 400".
   - Confirms. STK prompt on his phone. Enters PIN.
   - Within seconds: "Order 87AB confirmed. Thank you."
   - His WhatsApp also receives: "Your order is paid. Thank you from Kilimani Kitchen."

3. **Tenant View** (switch to dashboard tab):
   - Overview page: "1 order today, KSh 400 revenue, wallet KSh 400".
   - Click Orders: the new order is there.
   - Click Inbox: one WhatsApp message to Brian from the confirmation.
   - Click Wallet: one credit transaction.

4. **Isolation Check**:
   - Switch to Tenant B's dashboard.
   - Orders: empty.
   - Wallet: zero.
   - Customers: empty.
   - "Tenants are fully isolated. No code in the CRM filters by tenant id; Postgres RLS does it."

5. **Admin Scale Moment**:
   - "Now imagine this replicated across 500 Kenyan SMEs. Same codebase. Same database. Same deploy. Each tenant sees only their data."

Five minutes. Hit every point. Do not improvise.

---

## Final Polish Checklist

Before the demo:

- [ ] Three tenants seeded with data.
- [ ] Each tenant's USSD code registered in the AT simulator.
- [ ] Each tenant's WhatsApp sandbox phone is opted in.
- [ ] `mctaba-payments` configured with the shared sandbox credentials.
- [ ] All CRON jobs running.
- [ ] Outbox worker running and draining events.
- [ ] No `console.error` in a full flow run-through.
- [ ] Isolation test passing in CI.
- [ ] Screenshots captured for every step.

---

## Known Gaps For Week 30 To Cover

The demo works but these are not production-ready:

- **Monitoring.** Minimal logging, no alerting. Week 30 Day 1.
- **Documentation.** README is thin. Week 30 Day 2.
- **Real deploy.** Still on localhost. Week 30 Day 3.
- **Backups.** None. Week 30 Day 3.
- **TLS.** Localhost cert. Week 30 Day 3.
- **Rate limits beyond queue-level.** Week 30 Day 3.
- **Error reporting (Sentry etc).** Week 30 Day 3.

Do not try to fix these this week. Week 30 has its own plan.

---

## Week 29 Recap

### What you built
- USSD webhook routes by tenant based on service code.
- WhatsApp webhook routes by tenant based on phone_number_id.
- M-Pesa payments credit the correct tenant's wallet.
- Unified inbox showing all channels side by side.
- Reply-through-channel from the dashboard.
- A polished demo that touches every integration.

### Milestones hit
- Week 29 Day 1: USSD per tenant.
- Week 29 Day 2: WhatsApp per tenant.
- Week 29 Day 3: Payments per tenant.
- Week 29 Day 4: Unified inbox.
- Week 29 Day 5: End-to-end demo.

All milestones from the Week 27 plan for this week are done. You are on track.

---

## Peer Session

Swap demos with another student. Watch their demo. Take notes on:
- Clarity: could you understand what was happening?
- Completeness: did they cover every major piece?
- Polish: did anything look broken?
- Story: did they make the tenant pain believable?

Write a one-page peer review for each other.

---

## Weekend Prep

The last weekend before launch. Brief in `week29/weekend/project.md`. No new features.

Next week: test, document, deploy, rehearse, ship.
