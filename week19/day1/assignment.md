# Week 19 - Day 1 Assignment

## Title
Extract Your Payments Logic Into A Reusable Package

## Overview
Week 19 turns the payments code into a standalone, installable package. Today you extract it into `packages/mctaba-payments` in your monorepo, define the public API surface, and set up the package.json for publishing.

## Learning Objectives Assessed
- Design a clean public API for a shared library
- Set up a package.json for publishing
- Export a narrow, stable interface
- Keep internal implementation details private

## Prerequisites
- Week 18 completed

## AI Usage Rules

**Ratio this week:** 35% manual / 65% AI
**Habit:** You design the API, AI fills it in. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Filling in types and implementations after you designed the surface.
- **NOT ALLOWED FOR:** Designing the surface.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Design the public API

**What to do:**
In `api-design.md`, sketch the exports:

```javascript
// mctaba-payments/index.js
export { initiatePayment } from "./initiate";
export { verifyWebhook } from "./webhook";
export { issueRefund } from "./refund";
export { reconcile } from "./reconcile";
// NOT exported: token functions, internal HTTP clients
```

Specify each function's signature and return shape. This is the contract.

**Expected output:**
`api-design.md` committed.

### Task 2: Create the package folder

**What to do:**
```bash
mkdir -p packages/mctaba-payments/src
cd packages/mctaba-payments
npm init -y
```

Edit `package.json`:

```json
{
  "name": "@mctaba/payments",
  "version": "0.1.0",
  "main": "src/index.js",
  "type": "module",
  "files": ["src/"],
  "description": "Unified payments for M-Pesa, Stripe, and Airtel",
  "keywords": ["mpesa", "stripe", "airtel", "payments", "kenya"]
}
```

**Expected output:**
Package scaffolded.

### Task 3: Move provider code into the package

**What to do:**
Move `lib/mpesa.js`, `lib/stripe.js`, `lib/airtel.js`, and the unified `initiatePayment` from your Next.js app into `packages/mctaba-payments/src/providers/`.

In `src/index.js`:

```javascript
export { initiatePayment } from "./initiate.js";
export { verifyWebhook } from "./webhook.js";
// ...
```

Update imports in the Next.js app to `@mctaba/payments`.

**Expected output:**
Next.js app still works using the package.

### Task 4: Env variable injection

**What to do:**
Inside the package, do NOT read `process.env` directly. Instead, accept a config object at the top level:

```javascript
// src/index.js
import { createMpesa } from "./providers/mpesa.js";
import { createStripe } from "./providers/stripe.js";

export function createClient(config) {
  const mpesa = createMpesa(config.mpesa);
  const stripe = createStripe(config.stripe);
  // ...
  return { initiatePayment, verifyWebhook, issueRefund };
}
```

The consumer (Next.js app) creates one instance with their env vars:

```javascript
import { createClient } from "@mctaba/payments";
export const payments = createClient({
  mpesa: { consumerKey: process.env.MPESA_CONSUMER_KEY, /* ... */ },
  stripe: { secretKey: process.env.STRIPE_SECRET_KEY },
  airtel: { clientId: process.env.AIRTEL_CLIENT_ID, /* ... */ },
});
```

**Expected output:**
Package is env-variable-agnostic.

### Task 5: API design notes

**What to do:**
In `api-design.md`, explain your decisions:
- Why use a factory (`createClient`) instead of reading env vars?
- Why hide the provider-specific clients?
- What happens if a future version needs to add a new field?

Your own words.

**Expected output:**
Notes complete.

## Stretch Goals (Optional - Extra Credit)

- Add TypeScript types (even if you are staying on JS, type declarations are useful for consumers).
- Add a README with installation and usage examples.
- Add JSDoc on every exported function.

## Submission Requirements

- **What to submit:** Repo with package structure, `api-design.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 1

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| API design doc first | 15 | Committed before code moved. |
| Package folder + package.json | 15 | Ready for publishing. |
| Providers moved into package | 25 | Next.js still works. |
| Factory pattern with config injection | 25 | Package does not read process.env directly. |
| Design notes | 15 | Three questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Leaking `process.env` inside the library.** The library should not know about env vars. Consumers inject config.
- **Exporting internal functions "just in case".** Export the minimum. Add more later if needed.
- **Skipping the design doc.** The API is your product. Think before you ship.

## Resources

- Day 1 reading: [Extracting the Payments Service.md](./Extracting%20the%20Payments%20Service.md)
- Week 19 AI boundaries: [../ai.md](../ai.md)
