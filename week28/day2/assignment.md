# Week 28 - Day 2 Assignment

## Title
Auth Middleware And RLS Context

## Overview
Day 2 wires login, session tokens, and middleware that sets the Postgres RLS context on every request so every query is automatically tenant-scoped.

## Learning Objectives Assessed
- Login endpoint returning a session token
- Middleware that resolves tenant from subdomain
- Middleware that sets `app.tenant_id` on the DB session
- Deny any query that runs without tenant context

## Prerequisites
- Day 1 completed
- Week 27 RLS preview

## AI Usage Rules

**Ratio this week:** 40% manual / 60% AI
**Habit:** Capstone build. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Middleware boilerplate.
- **NOT ALLOWED FOR:** The tenant resolution logic -- this is tenant isolation, permanent rule.
- **AUDIT REQUIRED:** Yes. Tenant isolation audit required.

## Tasks

### Task 1: Login

**What to do:**
`POST /v1/login` resolves tenant by subdomain (from the Host header or a request-level middleware), finds the user within that tenant, verifies bcrypt, returns a signed JWT:

```json
{ "sub": "user-uuid", "tid": "tenant-uuid", "role": "owner" }
```

Reject across tenants: if `acme.host` sends login for a `brio.host` user, it fails.

**Expected output:**
Login works per tenant.

### Task 2: Auth middleware

**What to do:**
`requireAuth` middleware:
- Reads the JWT from Authorization header or a cookie.
- Verifies signature.
- Sets `req.tenantId`, `req.userId`, `req.role`.
- Rejects mismatched tenant (JWT `tid` must match the request's subdomain tenant).

**Expected output:**
Middleware enforces.

### Task 3: RLS context middleware

**What to do:**
Wrap request handlers in a DB client that is scoped to the tenant:

```javascript
async function withTenant(req, res, next) {
  const client = await pool.connect();
  try {
    await client.query("SELECT set_config('app.tenant_id', $1, true)", [req.tenantId]);
    req.db = client;
    await next();
  } finally {
    client.release();
  }
}
```

Inside handlers, use `req.db.query(...)` instead of `pool.query(...)`.

**Expected output:**
Any handler that forgets `req.db` will fail RLS (by design).

### Task 4: Cross-tenant test

**What to do:**
Write a test:
- Log in as Acme owner.
- Try to fetch a booking belonging to Brio (by UUID).
- Expect 404 (because RLS hides it).

Run and confirm.

**Expected output:**
Test passes.

### Task 5: Tenant isolation audit

**What to do:**
`AI_AUDIT.md` Tenant Isolation section:
- List every handler.
- Does it use `req.db`? Yes/No.
- Does it set tenant_id anywhere manually (it shouldn't)?
- Re-read each line yourself.

**Expected output:**
Audit committed.

## Stretch Goals (Optional - Extra Credit)

- Refresh tokens with rotation.
- Logout endpoint that invalidates sessions.
- Admin role + policy.

## Submission Requirements

- **What to submit:** Repo, middleware, tests, audit, `AI_AUDIT.md`.
- **Deadline:** End of Day 2

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Login endpoint | 20 | Per tenant. |
| Auth middleware | 20 | JWT + tenant check. |
| RLS context middleware | 25 | `req.db` pattern. |
| Cross-tenant test | 25 | Passes. |
| Isolation audit | 5 | Every handler classified. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Using the raw pool inside a handler.** Bypasses RLS. Always `req.db`.
- **Trusting the JWT `tid` without cross-checking subdomain.** Both must agree.
- **Forgetting `set_config` after releasing the client.** Use `true` (local) so the setting resets.

## Resources

- Day 2 reading: [Auth Middleware and RLS Context.md](./Auth%20Middleware%20and%20RLS%20Context.md)
- Week 28 AI boundaries: [../ai.md](../ai.md)
