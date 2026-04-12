# Week 16 Weekend: Project 3 -- E-commerce Lite + WhatsApp Updates

The third roadmap project. Build a complete, shippable e-commerce store where a customer can browse, pay with M-Pesa, and receive WhatsApp updates at every stage of the lifecycle. By Monday this project should be live, shareable, and demoable in three minutes.

**Estimated time:** 10-14 hours.

**Deadline:** Monday morning.

---

## Functional Requirements

### Storefront
- Minimum 15 real products in 3+ categories, with real photos.
- Catalogue, search, category filtering all work.
- Mobile-responsive.
- Shareable product URLs with Open Graph previews.

### Checkout
- Add to cart with persisted Zustand store.
- Three-step checkout: info -> payment -> confirmation.
- State machine prevents skipping steps.
- Cart cleared on confirmation.
- Works with JavaScript disabled for the form submissions (progressive enhancement bonus).

### Payments
- **M-Pesa STK Push** triggered from the Server Action.
- **Callback handling** with idempotency and amount verification.
- **Retry flow** for failed payments with 30-second rate limit.
- **Cash on delivery** path with its own lifecycle.
- **Admin reconciliation** button using the Query API.
- **Airtel Money stub** (not implemented, just a visible option with a "coming soon" message).

### Notifications
- **Customer confirmation** WhatsApp on payment success.
- **Owner notification** WhatsApp on every new paid order.
- **Lifecycle updates** at `shipped` and `delivered`.
- **Reply threading** from order confirmations back into the order's messages view.

### Admin
- Login with HttpOnly JWT cookie.
- Orders dashboard with status filter, detail page, state-machine updates.
- Cancellation restocks inventory.
- Product create/edit form (bonus).
- Chime on new order (bonus).

### Tracking
- `/track/[shortId]` public page for customers who lost their original confirmation.

---

## Technical Requirements

1. Next.js 14 App Router with Server Components and Server Actions everywhere.
2. Postgres + Redis shared between Next.js and Express.
3. All mutations go through Server Actions or explicit API routes.
4. All multi-row writes happen inside transactions with `FOR UPDATE` where concurrency matters.
5. Prices, amounts, and totals are always computed on the server.
6. Webhook idempotency on the M-Pesa callback.
7. Fire-and-forget notifications; payment callback responds in <500ms.
8. `db/schema.sql` and `db/seed.sql` check in and work on a fresh database.
9. `.env.example` is committed; secrets are not.
10. `README.md` includes: setup, env vars, a labelled screenshot of each major screen, and a "How to test end-to-end" section.

---

## Grading Rubric (100 pts)

| Area | Points |
|---|---|
| Working storefront (browse + detail + cart) | 10 |
| Working checkout with state machine | 10 |
| M-Pesa STK Push end-to-end | 20 |
| M-Pesa callback idempotency + amount verify | 10 |
| WhatsApp confirmation + reply threading | 15 |
| Admin dashboard + state transitions | 10 |
| Restock on cancel | 5 |
| Public tracking page | 5 |
| README + screenshots | 5 |
| Code cleanliness (no inline SQL in controllers, no hardcoded env access outside config) | 10 |

90+ is an A. 70-89 is a pass. Below 70 is a rewrite.

---

## Submission Checklist (for Monday demo)

- [ ] Live URL (Vercel for the shop, ngrok for the Express server is acceptable for demo).
- [ ] Walk through as a customer: browse -> add -> checkout -> pay -> receive WhatsApp.
- [ ] Walk through as an admin: login -> orders list -> detail -> mark shipped -> customer receives shipped message.
- [ ] Demonstrate a cancelled payment and retry.
- [ ] Demonstrate the tracking page with the short id from the WhatsApp confirmation.
- [ ] Show a reply to the confirmation message being threaded into the admin order detail.
- [ ] Explain, in 30 seconds, the idempotency strategy on the M-Pesa callback.

---

## Hints

**Deploy early.** Deploy the shop to Vercel on Saturday morning, before any real features are done. Fix the deploy errors first (env vars, image domains, build output). By Sunday night you are only pushing feature commits, not fighting infra.

**Use a real Kenyan phone for the demo.** Test numbers are fine for dev, but a live demo where you hold up your actual phone and a real STK prompt appears is the moment that sells Project 3 to the room. Use a live Daraja app (not sandbox) if you can get one approved; otherwise the sandbox with a registered test number still looks real.

**WhatsApp sandbox opt-in.** Every reviewer who tests your demo has to opt in to your test number first. Put the opt-in instructions prominently in the README: "Send 'join xxxx' to +14155238886" or whatever your sandbox phrase is.

**Screenshots, not live tests.** For peer grading, a screenshot of the WhatsApp message arriving is acceptable in place of a live test. Record a short screen capture of the M-Pesa prompt appearing on your phone too.

**Idempotency test.** POST the same callback body to your endpoint twice with `curl`. Screenshot the logs showing "Duplicate callback, already processed" the second time. This goes in the README under "Proving idempotency".

**The last 10%.** Project 3's grade is determined by the last 10% -- the polish. Empty states, error messages, loading spinners, success animations, clean URLs. Do not skip them.

**What NOT to do.** Do not integrate Airtel for real. Do not add Telegram bots. Do not build an analytics dashboard. Do not write end-to-end tests. Those are later weeks. Ship the one thing the brief asks for.

---

## If You Finish Early

- **A small product admin form** that lets you add a new product via the admin dashboard (Server Action, no file upload yet).
- **A "featured products" flag** and a home page hero section that highlights them.
- **Back-in-stock alerts** -- customers enter their phone, get a WhatsApp when stock > 0. One table, one cron trigger (we teach cron formally in Week 21, but `node-cron` is one npm install away).
- **Multi-language storefront.** English and Kiswahili product descriptions, `?lang=sw` query param. Use the same `t()` pattern from USSD.

Good luck. Project 3 is the first thing you can legitimately put on a CV. Next week we add the second payment provider so the shop can take payments from customers anywhere in the world.
