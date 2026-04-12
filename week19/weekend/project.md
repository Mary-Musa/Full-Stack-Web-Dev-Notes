# Week 19 Weekend: Demo App Using Your Published Package

Build a standalone demo that uses `@yourscope/payments` from npm, end-to-end, with no shared code with your main shop. If your package has bugs, the demo exposes them -- and you must fix them before Monday.

**Estimated time:** 4-5 hours.

**Deadline:** Monday morning.

---

## Pick A Use Case

Not an e-commerce shop. A single-purpose tool.

Suggestions:
- **Tip jar** -- one page, amount input, pay button.
- **Paid contact form** -- customer pays KSh 500 to send you a message.
- **Donation page** -- pick a real charity, donate via M-Pesa.
- **Event ticket** -- one event, fixed price, collects name/phone, delivers QR code.
- **Paid consulting booking** -- pay to book a 30-minute slot on your calendar.

The simpler, the better. The goal is to prove your package works in a context it was not built around.

---

## Requirements

1. Separate git repository. Not in your main shop folder.
2. Uses `@yourscope/payments` from npm (not `file:` or git).
3. Has its own Postgres database (or uses yours with separate tables -- isolate).
4. Takes at least one real sandbox payment through one provider.
5. Sends a WhatsApp confirmation (via the Week 11 bot or AT SMS, your choice).
6. Has a simple home page explaining the use case.
7. Deploys to Vercel or Railway.

---

## Technical Requirements

- Next.js 14 App Router (to stay consistent with the course).
- One page, one Server Action that calls `payments.initiate(...)`.
- Webhook route that calls `payments.handleCallback(...)`.
- Database migration with the tables the package needs (copy from the package's `sql/schema.sql`).
- `README.md` with how to run it locally and how to test.

---

## Deliverables

- [ ] Git repo with the demo code.
- [ ] Live URL of the deployed app.
- [ ] Screenshot of a completed test payment.
- [ ] A short list of any issues you found in your package while integrating, and the fixes you shipped as `0.1.1`.

---

## Grading Rubric (50 pts)

| Area | Points |
|---|---|
| Package used from the real registry, not file: | 10 |
| Payment flow works end-to-end | 15 |
| Webhook handled correctly | 10 |
| Deployed and shareable URL | 10 |
| README with setup and test instructions | 5 |

---

## Hints

**Use the package the "dumb" way.** Do not look at the internals. Treat it like a third-party library. If you catch yourself opening `node_modules/@yourscope/payments/...`, you are cheating.

**Find at least one bug.** I guarantee your package has bugs you did not notice because you wrote it around your shop's conventions. The demo will expose them. Common ones: a required env var with no validation message, a confusing error shape, a missing feature the README implied existed. Fix the bugs, republish, update your demo to use the new version.

**Keep the demo under 300 lines total.** If it is bigger, you are doing too much. The demo is a dogfooding exercise, not a new product.

**Deploy early, iterate often.** Push the empty shell to Vercel first. Add features. Each push deploys automatically. You learn the full cycle in compressed form.

Good luck. Next week is Telegram bots -- a new channel to add to your toolbox.
