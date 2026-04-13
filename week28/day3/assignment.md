# Week 28 - Day 3 Assignment

## Title
Products And Orders CRUD

## Overview
Day 3 builds the core CRUD for the capstone: whatever "product" and "order" mean in your vertical (bookings/slots, subscriptions, tickets...). Every handler uses `req.db`. Every query is RLS-scoped.

## Learning Objectives Assessed
- Build full CRUD for two resources
- Paginate listings with cursor
- Validate input with zod
- Follow the error envelope

## Prerequisites
- Days 1-2 completed

## AI Usage Rules

**Ratio this week:** 40% manual / 60% AI
**Habit:** Capstone build. See [../ai.md](../ai.md).

- **ALLOWED FOR:** CRUD scaffolding.
- **NOT ALLOWED FOR:** Cross-tenant guarantees.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Products endpoints

**What to do:**
Implement:
- `GET /v1/products` -- cursor pagination
- `POST /v1/products`
- `GET /v1/products/:id`
- `PATCH /v1/products/:id`
- `DELETE /v1/products/:id`

All use `req.db`. All validate with zod.

**Expected output:**
Five endpoints work.

### Task 2: Orders endpoints

**What to do:**
Same shape for orders. An order belongs to a product, has a status, a customer, and an amount.

```
POST /v1/orders
GET  /v1/orders/:id
PATCH /v1/orders/:id   // only status transitions
GET  /v1/orders?cursor=...
```

State transitions: `created -> paid -> fulfilled -> closed` (plus `cancelled` from anywhere).

**Expected output:**
Endpoints work.

### Task 3: Validation errors

**What to do:**
Every validation failure returns the error envelope with `code: "validation"` and a `details` object listing each field. No AI on the failure shape -- consistency matters.

Example:

```json
{
  "error": {
    "code": "validation",
    "message": "Request validation failed",
    "details": { "name": "required", "priceCents": "must be positive integer" }
  }
}
```

**Expected output:**
Validation consistently errors with the envelope.

### Task 4: Pagination test

**What to do:**
Insert 30 products. Paginate with `limit=10` three times and confirm all 30 are returned once, in order.

**Expected output:**
Test passes.

### Task 5: Cross-tenant test (repeat)

**What to do:**
Fetch a product from Acme using Brio's JWT. Expect 404. Same for orders.

**Expected output:**
Test passes.

## Stretch Goals (Optional - Extra Credit)

- Optimistic concurrency with an `updated_at` check.
- Soft delete.
- Bulk import endpoint.

## Submission Requirements

- **What to submit:** Repo, handlers, tests, `AI_AUDIT.md`.
- **Deadline:** End of Day 3

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Products CRUD | 30 | Five endpoints, `req.db` everywhere. |
| Orders CRUD | 30 | With valid transitions. |
| Validation envelope | 15 | Consistent. |
| Pagination test | 10 | Passes. |
| Cross-tenant test | 10 | Passes. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Pool.query shortcut.** Always `req.db`.
- **Transitioning order status arbitrarily.** Enforce the state machine.
- **Validation inside handlers by hand.** Use zod from the start.

## Resources

- Day 3 reading: [Products and Orders CRUD.md](./Products%20and%20Orders%20CRUD.md)
- Week 28 AI boundaries: [../ai.md](../ai.md)
