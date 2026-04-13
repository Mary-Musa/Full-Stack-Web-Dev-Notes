# Week 28 - Day 5 Assignment

## Title
Dashboard Shell, Week 28 Demo, And Git Hooks Redux

## Overview
Day 5 ships a dashboard shell (Next.js) for authenticated users of a tenant, runs a Week 28 demo, and adds a `pre-push` hook that runs the isolation test suite so you never push a regression.

## Learning Objectives Assessed
- Build an authenticated Next.js shell for a tenant
- Resolve the tenant from subdomain in Next.js middleware
- Add a `pre-push` git hook
- Present a short demo

## Prerequisites
- Days 1-4 completed

## AI Usage Rules

**Ratio this week:** 40% manual / 60% AI
**Habit:** Capstone build. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Dashboard scaffolding.
- **NOT ALLOWED FOR:** Auth middleware.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Tenant resolver middleware

**What to do:**
Next.js `middleware.ts` reads the subdomain from the Host header, looks up the tenant, and passes it in a header to pages:

```typescript
export async function middleware(req: NextRequest) {
  const host = req.headers.get("host") || "";
  const sub = host.split(".")[0];
  const tenant = await getTenantBySubdomain(sub);
  if (!tenant) return NextResponse.redirect(new URL("/not-found", req.url));
  const res = NextResponse.next();
  res.headers.set("x-tenant-id", tenant.id);
  return res;
}
```

**Expected output:**
Subdomain routing works.

### Task 2: Authed shell

**What to do:**
`app/(authed)/layout.js` wraps the dashboard. Server-side, read session cookie, verify, or redirect to `/login`.

Render a simple sidebar: Dashboard, Products, Orders, Wallet, Settings.

**Expected output:**
Logged-in view.

### Task 3: Dashboard home

**What to do:**
Display:
- Today's orders count
- Wallet balance
- Latest 5 products

All via Server Components using `req.db`.

**Expected output:**
Real data rendered.

### Task 4: pre-push hook

**What to do:**
`.husky/pre-push`:

```bash
npm run test:isolation
```

Package.json:

```json
"test:isolation": "vitest run tests/isolation.test.js"
```

Try to push while the isolation test fails -- confirm the hook blocks.

**Expected output:**
Push blocked on failing isolation.

### Task 5: Week 28 demo and reflection

**What to do:**
Record a 3-minute screen demo (or write a transcript) that covers:
- Signup and subdomain
- Login
- Cross-tenant leak test passing
- Dashboard shell

Save as `demo-week28.md` or `demo-week28.mp4`.

**Expected output:**
Artifact committed.

## Stretch Goals (Optional - Extra Credit)

- Theme toggle per tenant.
- Custom favicon per tenant.
- Feature flags table per tenant.

## Submission Requirements

- **What to submit:** Repo, hook, demo, `AI_AUDIT.md`.
- **Deadline:** End of Day 5

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Tenant resolver | 20 | Subdomain routing. |
| Authed shell | 20 | Redirect + sidebar. |
| Dashboard home | 25 | Real data. |
| pre-push hook | 20 | Blocks on failure. |
| Demo | 10 | Covers four topics. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Trusting `x-tenant-id` header from the user.** Only set it from trusted middleware.
- **Bypassing pre-push with --no-verify.** Do not.
- **Dashboard reading from `pool.query` directly.** Always through the req-scoped client.

## Resources

- Day 5 reading: [Dashboard Shell and Week 28 Demo.md](./Dashboard%20Shell%20and%20Week%2028%20Demo.md)
- Week 28 AI boundaries: [../ai.md](../ai.md)
