# Week 19 - Day 2 Assignment

## Title
Write A Package README And Type Declarations

## Overview
Today you write the README that any future user of your package will read, plus TypeScript type declarations (`.d.ts`) so consumers in TypeScript projects get autocomplete even though the package is in plain JS.

## Learning Objectives Assessed
- Write a package README that includes installation, usage, and examples
- Write a `.d.ts` file for JS code
- Document the public API

## Prerequisites
- Day 1 completed

## AI Usage Rules

**Ratio:** 35/65. **Habit:** You design the API, AI fills it in. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Prose drafting for the README.
- **NOT ALLOWED FOR:** Making up features that do not exist.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: README

**What to do:**
Create `packages/mctaba-payments/README.md`:

```markdown
# @mctaba/payments

Unified payments client for M-Pesa, Stripe, and Airtel.

## Installation

\```bash
npm install @mctaba/payments
\```

## Quick start

\```javascript
import { createClient } from "@mctaba/payments";

const payments = createClient({
  mpesa: {
    consumerKey: process.env.MPESA_CONSUMER_KEY,
    consumerSecret: process.env.MPESA_CONSUMER_SECRET,
    shortcode: process.env.MPESA_SHORTCODE,
    passkey: process.env.MPESA_PASSKEY,
    env: "sandbox",
  },
  stripe: { secretKey: process.env.STRIPE_SECRET_KEY },
  airtel: { /* ... */ },
});

const result = await payments.initiatePayment({
  provider: "mpesa",
  orderId: "abc-123",
  amountCents: 150000,
  phone: "254712345678",
});
\```

## API

### createClient(config)
Returns a configured client instance.

### initiatePayment(args)
...

## Contributing
...
```

The README must include: installation, a quick-start code block, an API reference, and a link to contributing.

**Expected output:**
README committed.

### Task 2: Type declarations

**What to do:**
Create `packages/mctaba-payments/src/index.d.ts`:

```typescript
export interface InitiatePaymentArgs {
  provider: "mpesa" | "stripe" | "airtel";
  orderId: string;
  amountCents: number;
  phone?: string;
  email?: string;
}

export interface InitiatePaymentResult {
  success: boolean;
  reference?: string;
  redirectUrl?: string;
  error?: string;
}

export interface Config {
  mpesa?: MpesaConfig;
  stripe?: StripeConfig;
  airtel?: AirtelConfig;
}

export interface MpesaConfig {
  consumerKey: string;
  consumerSecret: string;
  shortcode: string;
  passkey: string;
  env: "sandbox" | "production";
}

// ... etc

export function createClient(config: Config): {
  initiatePayment(args: InitiatePaymentArgs): Promise<InitiatePaymentResult>;
  verifyWebhook(provider: string, body: string, signature: string): boolean;
  issueRefund(orderId: string, reason?: string): Promise<any>;
};
```

Reference the type file in `package.json`:

```json
{
  "types": "src/index.d.ts"
}
```

**Expected output:**
Type declarations committed.

### Task 3: Test consumer

**What to do:**
In your Next.js app, import from `@mctaba/payments`. Your editor should now show autocomplete and type hints based on the `.d.ts`.

**Expected output:**
Editor hints work.

### Task 4: Short usage notes

**What to do:**
In `usage-notes.md`, write 5-7 sentences:
- Why write a README for a library only you use?
- Why write type declarations if the package is plain JS?
- Who is the audience for each document?

**Expected output:**
`usage-notes.md` committed.

### Task 5: Versioning plan

**What to do:**
In `versioning-plan.md`, write what constitutes a MAJOR, MINOR, and PATCH bump in your package. Specifically:

- MAJOR: breaking change to the `createClient` or `initiatePayment` signature.
- MINOR: new provider added. New optional arg.
- PATCH: bug fix. Docs update.

**Expected output:**
Committed.

## Stretch Goals (Optional - Extra Credit)

- Generate a proper API reference with a doc generator (JSDoc, TypeDoc).
- Set up prettier/eslint config for the package.
- Add badges to the README (npm version, build status).

## Submission Requirements

- **What to submit:** Repo with README, `.d.ts`, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 2

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| README with all required sections | 25 | Installation, quick start, API, contributing. |
| Type declarations file | 25 | Matches the JS runtime. Autocomplete works in a TS project. |
| Type-hint test in consumer | 15 | Editor shows hints. |
| Usage notes | 15 | Three questions answered. |
| Versioning plan | 15 | MAJOR/MINOR/PATCH defined for this package. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **README says the package can do things it cannot.** Match the README to reality.
- **`.d.ts` that drifts from the JS.** Every time you change the API, update both.
- **Publishing without a README.** npm will show "No description provided" -- embarrassing.

## Resources

- Day 2 reading: [Public API and README.md](./Public%20API%20and%20README.md)
- Week 19 AI boundaries: [../ai.md](../ai.md)
