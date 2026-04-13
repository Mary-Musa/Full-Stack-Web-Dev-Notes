# Week 29, Day 1: USSD Per Tenant

> **AI boundaries this week:** 25% manual / 75% AI. Habit: *AI is your hands; you are the brain.* See [ai.md](../ai.md).

By the end of today, a customer dialling a USSD code is routed to the right tenant's menu based on the service code, and the menus read from that tenant's products and customers.

**Prior-week concepts you will use today:**
- USSD state machines (Week 13)
- Per-tenant data with RLS (Week 28)
- `mctaba-payments` package for payments

**Estimated time:** 4 hours.

---

## How Tenants Get Short Codes

In production: each SME buys or rents a short code from Africa's Talking and tells you "add this to my tenant". In the Capstone: hardcode one shared sandbox code that routes to whichever tenant based on a query string or metadata.

For the demo, the simplest workable pattern:

- Africa's Talking sandbox code: `*384*1234#`.
- Each tenant has an extension: `*384*1234*1#` for Tenant 1, `*384*1234*2#` for Tenant 2.
- The `serviceCode` field on the USSD callback distinguishes them.

Alternatively, use subpaths on the webhook URL. Africa's Talking lets you set different callback URLs per short code; you register one per tenant.

Store the mapping in the tenant row:

```sql
-- Already in the schema:
tenants.ussd_code TEXT UNIQUE
```

Populate during signup or via a settings page.

---

## The Tenant Resolver

```javascript
// api/services/ussd/tenantResolver.js
const { query } = require("../../config/db");

async function resolveTenantFromUssd({ serviceCode }) {
  // Try exact match
  const { rows } = await query(
    "SELECT * FROM tenants WHERE ussd_code = $1 AND status = 'active'",
    [serviceCode]
  );
  return rows[0];
}

module.exports = { resolveTenantFromUssd };
```

Returns the tenant row or null. The caller handles the null case with a fallback menu.

---

## The USSD Webhook

```javascript
// api/routes/webhooks/ussd.routes.js
const express = require("express");
const router = express.Router();
const { resolveTenantFromUssd } = require("../../services/ussd/tenantResolver");
const { pool } = require("../../config/db");
const ussdDispatcher = require("../../services/ussd/dispatcher");

router.post("/", express.urlencoded({ extended: false }), async (req, res) => {
  const { sessionId, serviceCode, phoneNumber, text } = req.body;

  const tenant = await resolveTenantFromUssd({ serviceCode });
  if (!tenant) {
    return res.set("Content-Type", "text/plain").send(
      "END This service code is not set up. Contact support."
    );
  }

  const client = await pool.connect();
  try {
    await client.query(`SET LOCAL app.current_tenant = $1`, [tenant.id]);

    const response = await ussdDispatcher.run({
      sessionId,
      phoneNumber,
      rawText: text || "",
      tenant,
      client,
    });

    res.set("Content-Type", "text/plain").send(response);
  } catch (err) {
    console.error("USSD dispatcher error:", err);
    res.set("Content-Type", "text/plain").send("END Something went wrong. Please try again.");
  } finally {
    client.release();
  }
});

module.exports = router;
```

Two things worth noting.

**RLS context is set per-request.** The dispatcher and any SQL it runs will only see this tenant's data.

**Client is passed through** to the dispatcher so state-machine handlers can use the same connection and stay inside the tenant's RLS context.

---

## Tenant-Aware Menus

The dispatcher runs state handlers like Week 13. Each handler receives the client and can query:

```javascript
// api/services/ussd/states/browse_products.js
module.exports = async function browseProducts({ input, context, tenant, client }) {
  // The RLS context is already set, so this query is tenant-scoped
  const { rows: products } = await client.query(
    "SELECT id, name, price_cents FROM products WHERE stock > 0 ORDER BY name LIMIT 5"
  );

  if (products.length === 0) {
    return {
      response: `END ${tenant.name} has no products in stock.`,
      nextState: "done",
      nextContext: {},
    };
  }

  if (input === "") {
    const menu = products.map((p, i) =>
      `${i + 1}. ${p.name} - KSh ${(p.price_cents / 100).toLocaleString()}`
    ).join("\n");
    return {
      response: `CON ${tenant.name}\n${menu}\n0. Back`,
      nextState: "browse_products",
      nextContext: { ...context, products: products.map((p) => p.id) },
    };
  }

  if (input === "0") {
    return { response: "CON Back to main menu\n1. Browse\n2. My orders", nextState: "welcome", nextContext: {} };
  }

  const index = parseInt(input, 10) - 1;
  if (isNaN(index) || index < 0 || index >= products.length) {
    return { response: "CON Invalid option. Pick 1-5 or 0 for back.", nextState: "browse_products", nextContext: context };
  }

  const selected = products[index];
  return {
    response: `CON ${selected.name} - KSh ${(selected.price_cents / 100).toLocaleString()}\n1. Buy now\n0. Back`,
    nextState: "confirm_purchase",
    nextContext: { ...context, selectedProductId: selected.id },
  };
};
```

Notice: the handler reads products without any `WHERE tenant_id =`. RLS is doing the work. The `tenant` parameter is available for display purposes (the tenant's name in the menu).

---

## Session State

Session state in Redis is keyed by session id + tenant id (to avoid collisions across tenants using the same session id pool):

```javascript
const sessionKey = (sessionId, tenantId) => `ussd:session:${tenantId}:${sessionId}`;
```

The Redis is shared across tenants but the key namespace isolates them.

---

## Testing The Flow

1. Make sure Tenant 1 has at least three products.
2. Make sure Tenant 1's `ussd_code` is `*384*1234*1#`.
3. Dial the code from the Africa's Talking simulator.
4. See the menu with Tenant 1's name and products.
5. Pick a product. See the confirm prompt.
6. Pick Buy. (Payment wiring tomorrow.)

Compare to Tenant 2 at `*384*1234*2#` -- different menu, different products.

---

## Checkpoint

1. The USSD webhook routes to the right tenant based on service code.
2. Dialling from an unknown code returns a friendly error.
3. The menus show the tenant's name and its products.
4. Two tenants with different codes show different menus.
5. Session state does not leak across tenants.

Commit:

```bash
git add .
git commit -m "feat: per-tenant ussd routing with rls context"
```

---

## What You Learned

- USSD routing is "look up by service code" not "look up by tenant id in a cookie".
- Setting the RLS context in the webhook handler propagates through every query.
- State handlers do not need tenant awareness once RLS is set.

Tomorrow: WhatsApp per tenant.
