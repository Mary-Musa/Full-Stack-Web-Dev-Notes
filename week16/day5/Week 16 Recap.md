# Week 16, Day 5: Week Recap and Project 3 Ready-Up

This is the week you shipped Project 3 in all but name. The weekend is the final push. Every piece is in place: catalogue, cart, checkout, M-Pesa, WhatsApp, admin, customer tracking.

---

## What You Built This Week

1. **M-Pesa STK Push** from the Next.js shop via the Express payments service.
2. **M-Pesa callback** with idempotency, amount verification, failure code mapping.
3. **Admin reconciliation** via Daraja's STK Push Query.
4. **WhatsApp order confirmations** with template and sandbox paths.
5. **Reply threading** on the confirmation message.
6. **Owner notifications** for new orders.
7. **Chime alert** on the admin dashboard.
8. **Retry flow** for failed payments with rate limiting.
9. **Lifecycle WhatsApp updates** at every state transition.
10. **COD shortcut path** through the state machine.
11. **Public order tracking** by short id.
12. **Empty-state UX** on admin pages.

You now have a complete working e-commerce flow. The only thing missing is design polish.

---

## Self-Review Questions

**M-Pesa**
1. Why do we route M-Pesa through the Express CRM server instead of calling Daraja from Next.js directly?
2. What does `CheckoutRequestID` do in the payment flow?
3. Why is the callback handler idempotent?
4. Why do we verify the amount in the callback, not trust Daraja's "success"?
5. When would you use the STK Push Query API?

**WhatsApp**
6. What is the difference between a session message and a template message?
7. Why is `order_confirmation` a Utility template, not Marketing?
8. How do replies to order confirmations get threaded back to the order?
9. Why is notification sending always fire-and-forget from the payment callback?

**Architecture**
10. What are the two separate servers in the Project 3 architecture and what does each own?
11. Why does the order exist before payment is confirmed?
12. What does rate-limiting retries protect you from?

**Lifecycle**
13. What are the states an order moves through, and which ones fire WhatsApp messages?
14. How does a COD order transition differently from an M-Pesa order?
15. What happens on the admin side when a new paid order lands?

Target: 12/15. Anything less means go back to the relevant day.

---

## Peer Coding Session

### Track A: Delivery-time estimation

Replace the hardcoded "2 days" delivery estimate with one computed from the delivery address. Hardcode a few Nairobi suburbs (`Westlands`, `Kilimani`, `Eastleigh`, etc.) with their estimated times. Anything else: "3-5 days".

### Track B: Refund support

Add a "Refund" button on the admin order detail for paid orders. The Server Action should trigger Daraja's "B2C" API to send money back to the customer's phone. This is non-trivial -- Daraja B2C requires additional configuration. Aim for a working skeleton, not a full implementation.

### Track C: Order packing slip

Generate a PDF packing slip from the admin order detail page. Reuse `pdfkit` from Week 10. The slip should have the order id as a QR code, customer info, and the item list. Print one per order.

### Track D: Discount codes

Add a `discount_codes` table with `code`, `percent_off`, `valid_until`, `usage_count`. Customer types the code on the payment page; server validates and adjusts the subtotal. One table, one Server Action, one form field.

---

## Weekend Prep -- Project 3 is the big one

Project 3 is the first truly end-to-end product you will ship in this Marathon. Read the brief carefully before Monday -- it has specific deliverables the earlier weekend projects did not.

Three questions to sleep on:

1. **What is your niche?** You picked one in Week 14 for the catalogue MVP. Does it still make sense for a full shop? If not, pivot now. If yes, commit -- no more changing.
2. **What payment methods will you actually support?** M-Pesa is mandatory; COD is recommended; Airtel is optional for now and ships fully in Week 17.
3. **Who is your demo audience?** Your peer partner, your cohort, the instructor -- or someone outside the class? An outside demo raises the bar on polish.

Project 3 is demo-graded next Monday. Bring a real phone, a real number, a real product list, and a three-minute story.

Next week we double up on payments: Stripe for global customers, Airtel for local alternatives. Then Week 18 we harden the whole payment layer for real life.
