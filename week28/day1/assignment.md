# Week 28 - Day 1 Assignment

## Title
Tenant Signup -- Create Tenant, Owner, Subdomain

## Overview
Week 28 is the multi-tenant core. Today you build tenant signup: a public form that creates a tenant row, its first owner user, and a unique subdomain. No stealing subdomains; no globally unique emails.

## Learning Objectives Assessed
- Create a tenant + owner atomically
- Enforce subdomain uniqueness
- Hash passwords
- Return the right next step to the user

## Prerequisites
- Week 27 completed

## AI Usage Rules

**Ratio this week:** 40% manual / 60% AI
**Habit:** Capstone build -- you own correctness, AI handles the repetition. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Form scaffolding, validation helpers.
- **NOT ALLOWED FOR:** Auth logic -- this is permanent-rule territory.
- **AUDIT REQUIRED:** Yes. Auth audit required.

## Tasks

### Task 1: Signup endpoint

**What to do:**
`POST /v1/signup` accepts:

```json
{
  "tenantName": "Acme Stylists",
  "subdomain": "acme",
  "ownerEmail": "owner@acme.co.ke",
  "ownerPassword": "string at least 10 chars"
}
```

In one transaction:
- Check subdomain is available and matches `^[a-z0-9-]{3,30}$`.
- Insert `tenants` row.
- Insert `users` row with role `owner`, password hashed with bcrypt.
- Return `{ tenantId, subdomain }`.

Reject 409 on taken subdomain.

**Expected output:**
Signup works.

### Task 2: Reserved subdomains

**What to do:**
Block `www`, `app`, `admin`, `api`, `staging`, etc. List them in a constant. Return 422 if attempted.

**Expected output:**
Blocked.

### Task 3: Signup page

**What to do:**
Next.js `/signup` page with the form. Call the API. On success, redirect to `https://{subdomain}.yourdomain.dev/login`.

For local dev, document how to hit `http://acme.localhost:3000/login` in README.

**Expected output:**
End-to-end signup works.

### Task 4: Transaction test

**What to do:**
Write an integration test that simulates a crash between the tenant insert and the user insert. Verify the tenant row is rolled back (because of the transaction).

**Expected output:**
Test passes.

### Task 5: Auth audit

**What to do:**
`AI_AUDIT.md` gets an Auth Audit section listing every auth-touching file in the capstone so far:
- Password hashing
- Subdomain generation
- Signup handler

For each, mark `self` or `AI` and confirm you re-read each line yourself.

**Expected output:**
Audit committed.

## Stretch Goals (Optional - Extra Credit)

- Email verification before activation.
- Invite link flow for the first staff user.
- Welcome email template.

## Submission Requirements

- **What to submit:** Repo, signup flow, tests, audit, `AI_AUDIT.md`.
- **Deadline:** End of Day 1

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Signup endpoint | 30 | Transactional. |
| Reserved subdomains | 10 | Blocked. |
| Signup page + redirect | 20 | End-to-end. |
| Transaction test | 20 | Rollback verified. |
| Auth audit | 15 | Every file re-read. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Password in plaintext anywhere.** Including logs.
- **Subdomain collisions with existing routes.** `/api`, `/www` etc.
- **Signup without transaction.** Half-created tenant is worse than no tenant.

## Resources

- Day 1 reading: [Tenant Signup.md](./Tenant%20Signup.md)
- Week 28 AI boundaries: [../ai.md](../ai.md)
