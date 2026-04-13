# Week 19 - Day 3 Assignment

## Title
Test The Package With Vitest And Mock The Providers

## Overview
Today you write real tests for the payments package. You mock each provider's HTTP client so tests run without hitting the real APIs, then test the unified interface.

## Learning Objectives Assessed
- Write unit tests for a library package
- Mock external HTTP calls (axios, stripe, fetch)
- Test success and failure paths
- Achieve decent coverage without running real payments

## Prerequisites
- Days 1-2 completed

## AI Usage Rules

**Ratio:** 35/65. **Habit:** You design the API. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Test boilerplate after you named the scenarios.
- **NOT ALLOWED FOR:** Making up assertions that do not match real provider responses.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Install test tools

**What to do:**
In `packages/mctaba-payments`:

```bash
npm install -D vitest
```

Add test script:

```json
"scripts": { "test": "vitest run" }
```

**Expected output:**
`npm test` runs (no tests yet).

### Task 2: Test the unified interface

**What to do:**
Create `src/initiate.test.js`:

```javascript
import { describe, it, expect, vi } from "vitest";
import { createClient } from "./index.js";

describe("initiatePayment", () => {
  it("routes mpesa requests", async () => {
    const client = createClient({
      mpesa: { consumerKey: "x", consumerSecret: "y", shortcode: "1", passkey: "z", env: "sandbox" },
    });
    // Mock the HTTP call
    vi.mock("axios", () => ({
      default: { post: vi.fn().mockResolvedValue({ data: { CheckoutRequestID: "abc" } }) },
    }));

    const result = await client.initiatePayment({
      provider: "mpesa",
      orderId: "o1",
      amountCents: 100,
      phone: "254712345678",
    });

    expect(result.success).toBe(true);
    expect(result.reference).toBe("abc");
  });

  it("returns error for unknown provider", async () => {
    const client = createClient({});
    const result = await client.initiatePayment({
      provider: "unknown",
      orderId: "o1",
      amountCents: 100,
    });
    expect(result.success).toBe(false);
    expect(result.error).toMatch(/Unknown provider/);
  });
});
```

Extend with tests for Stripe and Airtel.

**Expected output:**
Tests pass.

### Task 3: Test error paths

**What to do:**
Add tests for:
- Network failure (provider throws)
- Invalid phone format (if your library validates)
- Missing config (should throw at createClient)

**Expected output:**
At least 3 error-path tests passing.

### Task 4: Coverage

**What to do:**
```bash
npx vitest run --coverage
```

Aim for at least 70% line coverage on the package. Add tests to fill gaps.

**Expected output:**
Coverage report. Screenshot `day3-coverage.png`.

### Task 5: CI workflow for the package

**What to do:**
Add a GitHub Actions workflow `.github/workflows/package-test.yml`:

```yaml
name: Package tests
on:
  pull_request:
    paths:
      - "packages/mctaba-payments/**"
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: cd packages/mctaba-payments && npm ci && npm test
```

Push. Verify it runs on your next PR touching the package.

**Expected output:**
Workflow green.

## Stretch Goals (Optional - Extra Credit)

- Add a "contract test" that verifies your `.d.ts` matches the runtime.
- Add snapshot tests for provider responses.
- Use `msw` (Mock Service Worker) instead of manual mocks.

## Submission Requirements

- **What to submit:** Repo with tests, coverage screenshot, workflow, `AI_AUDIT.md`.
- **Deadline:** End of Day 3

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Vitest set up | 10 | `npm test` works. |
| Happy-path tests for 3 providers | 30 | Each provider has at least one passing test. |
| Error-path tests | 20 | At least 3 failure scenarios covered. |
| Coverage >= 70% | 15 | Screenshot confirms. |
| CI workflow for package | 20 | Runs on PR. Green. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Testing the mock instead of the code.** Assert on your code's behaviour, not on "did the mock get called".
- **Hitting real APIs from tests.** Slow, fragile, costs money, fails when sandboxes are down. Always mock.
- **Chasing 100% coverage.** 70% covers the important paths. The last 30% is often untestable edge cases.

## Resources

- Day 3 reading: [Testing the Package.md](./Testing%20the%20Package.md)
- Week 19 AI boundaries: [../ai.md](../ai.md)
- Vitest docs: https://vitest.dev/
