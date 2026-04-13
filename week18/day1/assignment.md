# Week 18 - Day 1 Assignment

## Title
Webhook Signature Verification -- Stripe, Meta, and Daraja

## Overview
Week 18 is the security dip week. Today you lock down every webhook you have with proper signature verification and tighter security headers. Stripe sends HMAC-signed webhooks by default; Meta uses an HMAC SHA-256 header; Daraja does not sign, so you must rely on the checkout-id match. You will audit and harden all three.

## Learning Objectives Assessed
- Verify Stripe webhook signatures with the official library
- Verify Meta webhook signatures using crypto.createHmac
- Use timingSafeEqual to avoid timing-attack leaks
- Reject any webhook that fails verification

## Prerequisites
- Weeks 11 and 16-17 completed

## AI Usage Rules

**Ratio this week:** 45% manual / 55% AI (SECURITY DIP)
**Habit:** Audit every security line AI writes. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Scaffolding route structure.
- **NOT ALLOWED FOR:** Any signature verification line. Every one hand-written.
- **AUDIT REQUIRED:** Yes. Security audit section.

## Tasks

### Task 1: Stripe webhook verification

**What to do:**
Stripe requires you to verify using the raw request body (not the parsed JSON). In Next.js, this means using a special config for that route.

Create `app/api/stripe/webhook/route.js`:

```javascript
import { stripe } from "@/lib/stripe";
import { NextResponse } from "next/server";

export async function POST(req) {
  const body = await req.text(); // raw body
  const signature = req.headers.get("stripe-signature");

  let event;
  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET
    );
  } catch (err) {
    console.error("Invalid Stripe signature:", err.message);
    return NextResponse.json({ error: "Invalid signature" }, { status: 400 });
  }

  // Handle the event
  if (event.type === "checkout.session.completed") {
    const session = event.data.object;
    // Reconcile the order (similar to Week 16 Day 2)
    console.log("Stripe payment:", session.id);
  }

  return NextResponse.json({ received: true });
}
```

Set `STRIPE_WEBHOOK_SECRET` in env. Get it from the Stripe dashboard (Developers > Webhooks > Add endpoint).

Use the Stripe CLI to test locally: `stripe listen --forward-to http://localhost:3000/api/stripe/webhook`.

**Expected output:**
Stripe test events verify successfully.

### Task 2: Meta webhook re-verification

**What to do:**
Revisit your Week 11 Day 2 HMAC check. Make sure it is still in place and hand-written. Add `timingSafeEqual`:

```javascript
if (!crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
  throw new Error("Invalid signature");
}
```

If you did not already use `timingSafeEqual`, add it now.

**Expected output:**
Meta webhook still works. Timing-safe comparison in place.

### Task 3: Daraja IP allowlisting (defense in depth)

**What to do:**
Daraja does not sign callbacks, so you compensate with:
1. The checkout-id match you already built (Week 16 Day 2) -- keep it
2. IP allowlisting -- restrict the callback endpoint to Safaricom's IP ranges

Safaricom publishes their IP ranges in their docs. Implement a middleware:

```javascript
const SAFARICOM_IPS = ["196.201.214.200", /* ... */];

export function middleware(req) {
  const ip = req.headers.get("x-forwarded-for")?.split(",")[0] || req.ip;
  if (!SAFARICOM_IPS.includes(ip)) {
    return new Response("Forbidden", { status: 403 });
  }
}
```

For local dev (ngrok), skip the check when `NODE_ENV !== "production"`.

**Expected output:**
Production callbacks from non-Safaricom IPs are rejected.

### Task 4: Security test matrix

**What to do:**
Create `security-tests.md`. List every webhook and test each one with:
- Valid signature -> accepted
- Invalid signature (tampered body) -> rejected
- Missing signature header -> rejected
- Replay of a previous valid signature -> depends on your idempotency check

Test each case with curl or a small script. Record the actual status codes.

**Expected output:**
`security-tests.md` with real test results.

### Task 5: Audit section

**What to do:**
In `AI_AUDIT.md`, add a "Security audit per file" section. For every security-critical file, confirm:
- Who wrote each line
- When it was last reviewed
- Any AI involvement (should be zero)

**Expected output:**
Audit updated.

## Stretch Goals (Optional - Extra Credit)

- Add a request signing library and use it for your own internal API calls.
- Log every rejected webhook to a `security_events` table.
- Set up a security alert via email when N rejections happen in an hour.

## Submission Requirements

- **What to submit:** Repo, webhook routes, `security-tests.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 1

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Stripe webhook signature verify | 25 | Official library + raw body. |
| Meta webhook with timingSafeEqual | 20 | Hand-written. |
| Daraja IP allowlisting | 15 | Production-only. |
| Security test matrix | 25 | All webhooks tested for both success and failure. |
| Audit section | 10 | Manual-only confirmed. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Using `req.body` instead of the raw body for Stripe.** Stripe needs the untouched bytes. Next.js JSON parsing corrupts the signature.
- **`===` for signature comparison.** Timing leaks. Always `timingSafeEqual`.
- **Skipping the IP check because it is "annoying" in dev.** Apply only in production.

## Resources

- Day 1 reading: [Webhook Security.md](./Webhook%20Security.md)
- Week 18 AI boundaries: [../ai.md](../ai.md)
- Stripe webhook docs: https://stripe.com/docs/webhooks/signatures
