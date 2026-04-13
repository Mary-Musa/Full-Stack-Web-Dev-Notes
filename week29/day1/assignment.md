# Week 29 - Day 1 Assignment

## Title
USSD Per Tenant -- Shared Code, Per-Tenant Config

## Overview
Week 29 brings integrations into the capstone. Today is USSD: one shared USSD endpoint that routes per tenant based on the service code or a DB lookup.

## Learning Objectives Assessed
- Route incoming USSD by tenant
- Store per-tenant USSD configuration
- Reuse session code from Week 15
- Keep responses under 182 characters

## Prerequisites
- Week 28 completed
- Africa's Talking simulator account

## AI Usage Rules

**Ratio this week:** 25% manual / 75% AI
**Habit:** Integrations week -- AI does most, you verify contracts. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Menu boilerplate.
- **NOT ALLOWED FOR:** Tenant resolution.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: ussd_configs table

**What to do:**
```sql
CREATE TABLE ussd_configs (
  tenant_id UUID PRIMARY KEY REFERENCES tenants(id),
  service_code TEXT UNIQUE NOT NULL,
  welcome_text TEXT NOT NULL
);
```

Seed two tenants with different service codes.

**Expected output:**
Two configs present.

### Task 2: Shared USSD endpoint

**What to do:**
`POST /ussd` receives Africa's Talking payload. Identify the tenant by the service code:

```javascript
const { rows } = await pool.query("SELECT tenant_id FROM ussd_configs WHERE service_code = $1", [serviceCode]);
if (!rows[0]) return res.send("END Service unavailable");
req.tenantId = rows[0].tenant_id;
await client.query("SELECT set_config('app.tenant_id', $1, true)", [req.tenantId]);
```

Then run the menu logic within that tenant's data.

**Expected output:**
Two tenants, two service codes, each sees their own data.

### Task 3: Menu flow

**What to do:**
A minimal menu per tenant:

```
Welcome to {tenantName}
1. View products
2. Check balance
3. Exit
```

Options 1 and 2 query tenant-scoped tables.

**Expected output:**
Menu works for both tenants without leakage.

### Task 4: Cross-tenant test

**What to do:**
Script a fake USSD request to service code A but try to look up product data with a known Tenant B ID. The query should return nothing (RLS protects).

**Expected output:**
Test passes.

### Task 5: USSD quirks notes

**What to do:**
In `ussd-quirks.md`, 5-7 sentences:
- Why 182 characters?
- What happens if your response string is longer?
- How do you handle a user pressing back (`*`)?
- What is a `CON` vs `END` reply?

Your own words.

**Expected output:**
Committed.

## Stretch Goals (Optional - Extra Credit)

- Add language preference per tenant.
- Audit every menu option selected in a `ussd_events` table.
- Time out idle sessions at 60 seconds.

## Submission Requirements

- **What to submit:** Repo, endpoint, tests, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 1

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| ussd_configs | 15 | Two tenants seeded. |
| Shared endpoint routes | 30 | By service code. |
| Menu flow | 25 | Tenant-scoped. |
| Cross-tenant test | 20 | Passes. |
| USSD notes | 5 | Four questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Trusting the phone number as the tenant.** Same phone can belong to multiple tenants.
- **Service code leakage.** Never return tenant-specific data without setting tenant context.
- **Long USSD responses.** 182 cap.

## Resources

- Day 1 reading: [USSD Per Tenant.md](./USSD%20Per%20Tenant.md)
- Week 29 AI boundaries: [../ai.md](../ai.md)
