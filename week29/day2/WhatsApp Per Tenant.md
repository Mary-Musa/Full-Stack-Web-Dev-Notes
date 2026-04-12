# Week 29, Day 2: WhatsApp Per Tenant

By the end of today, a single WhatsApp webhook routes incoming messages to the right tenant based on the recipient phone number id, and a customer messaging any tenant's bot gets a tenant-specific response.

**Prior concepts:** yesterday's USSD routing; Week 11 WhatsApp bot.

**Estimated time:** 4 hours.

---

## How Meta Webhooks Route

Meta sends all webhooks for your app to one URL. The payload contains:

```json
{
  "entry": [{
    "changes": [{
      "value": {
        "metadata": {
          "phone_number_id": "123456789",
          "display_phone_number": "+254712000001"
        },
        "messages": [...]
      }
    }]
  }]
}
```

The `phone_number_id` identifies which of your registered numbers received the message. For a multi-tenant setup, each tenant is connected to a different phone_number_id. The webhook handler reads it and looks up the tenant.

In practice, you will have one Meta Business account with multiple WABA (WhatsApp Business Accounts) or multiple phone numbers under one WABA. The sandbox has one test number; in production you rent one per tenant or let them bring their own.

---

## Storing The Mapping

Already in the schema:

```sql
tenants.whatsapp_phone_number_id TEXT
```

Populate during tenant setup. A settings page lets the owner paste their phone number id from the Meta dashboard.

---

## The Webhook Handler

```javascript
// api/routes/webhooks/whatsapp.routes.js
const express = require("express");
const router = express.Router();
const { query, pool } = require("../../config/db");
const whatsappDispatcher = require("../../services/whatsapp/dispatcher");

// Verification (Meta initial handshake)
router.get("/", (req, res) => {
  const mode = req.query["hub.mode"];
  const token = req.query["hub.verify_token"];
  const challenge = req.query["hub.challenge"];
  if (mode === "subscribe" && token === process.env.META_VERIFY_TOKEN) {
    return res.send(challenge);
  }
  return res.sendStatus(403);
});

// Incoming message
router.post("/", express.json({
  verify: (req, _res, buf) => { req.rawBody = buf; }
}), async (req, res) => {
  res.sendStatus(200); // reply fast, process after

  const body = req.body;
  if (!verifySignature(body, req.headers["x-hub-signature-256"], req.rawBody)) {
    console.warn("Invalid WhatsApp signature");
    return;
  }

  const entries = body.entry || [];
  for (const entry of entries) {
    for (const change of entry.changes || []) {
      const value = change.value;
      const phoneNumberId = value?.metadata?.phone_number_id;
      if (!phoneNumberId) continue;

      const tenant = await findTenantByPhoneNumberId(phoneNumberId);
      if (!tenant) {
        console.warn(`No tenant for phone_number_id ${phoneNumberId}`);
        continue;
      }

      for (const message of value.messages || []) {
        await processMessage(tenant, message);
      }
    }
  }
});

async function findTenantByPhoneNumberId(phoneNumberId) {
  const { rows } = await query(
    "SELECT * FROM tenants WHERE whatsapp_phone_number_id = $1 AND status = 'active'",
    [phoneNumberId]
  );
  return rows[0];
}

async function processMessage(tenant, message) {
  const client = await pool.connect();
  try {
    await client.query(`SET LOCAL app.current_tenant = $1`, [tenant.id]);
    await whatsappDispatcher.run({ tenant, message, client });
  } catch (err) {
    console.error("WhatsApp dispatch error:", err);
  } finally {
    client.release();
  }
}

module.exports = router;
```

Two patterns to reuse everywhere.

**Respond 200 immediately.** Meta retries if you do not reply in 5 seconds. Process the message after sending the response, not before.

**RLS context set per message.** Each message inside a webhook call gets its own RLS context based on which tenant it belongs to. A webhook can legitimately contain messages for multiple tenants (rare but possible).

---

## The Dispatcher

```javascript
// api/services/whatsapp/dispatcher.js
const { query } = require("../../config/db");

async function run({ tenant, message, client }) {
  const from = message.from;
  const text = message.text?.body || "";

  // Upsert customer
  await client.query(
    `INSERT INTO customers (tenant_id, phone, first_seen_at)
     VALUES ($1, $2, NOW())
     ON CONFLICT (tenant_id, phone) DO NOTHING`,
    [tenant.id, from]
  );

  // Log inbound message
  await client.query(
    `INSERT INTO messages (tenant_id, customer_id, channel, direction, body, raw_payload)
     SELECT $1, id, 'whatsapp', 'in', $2, $3
     FROM customers WHERE tenant_id = $1 AND phone = $4`,
    [tenant.id, text, JSON.stringify(message), from]
  );

  // Handle the message
  if (text.toLowerCase().includes("order")) {
    await sendOrderMenu(tenant, from);
  } else {
    await sendWelcome(tenant, from);
  }
}

async function sendWelcome(tenant, to) {
  await sendWhatsAppText(tenant, to,
    `Karibu! This is ${tenant.name}. Reply "order" to start an order.`
  );
}

async function sendOrderMenu(tenant, to) {
  const { rows: products } = await query(
    `SELECT id, name, price_cents FROM products WHERE tenant_id = $1 AND stock > 0 LIMIT 5`,
    [tenant.id]
  );
  const list = products.map((p, i) =>
    `${i + 1}. ${p.name} - KSh ${(p.price_cents / 100).toLocaleString()}`
  ).join("\n");
  await sendWhatsAppText(tenant, to,
    `${tenant.name} products:\n${list}\nReply with the number you want to buy.`
  );
}

async function sendWhatsAppText(tenant, to, text) {
  const axios = require("axios");
  await axios.post(
    `https://graph.facebook.com/v21.0/${tenant.whatsapp_phone_number_id}/messages`,
    {
      messaging_product: "whatsapp",
      to,
      type: "text",
      text: { body: text },
    },
    { headers: { Authorization: `Bearer ${process.env.META_ACCESS_TOKEN}` } }
  );
}

module.exports = { run };
```

This is a toy bot. Real ones use the state-machine pattern from Week 11 with Redis-backed sessions. For the capstone demo, a simple "welcome -> order menu" is enough.

Note: `sendWhatsAppText` posts to the phone_number_id URL, so the message appears to come *from* that tenant's number. Meta handles the routing on their side.

---

## Token Per Tenant (Optional)

For the capstone, one Meta access token covers all phone numbers. For a production system where each tenant brings their own WABA, you would store a separate token per tenant and load it from the tenant row. Schema:

```sql
ALTER TABLE tenants ADD COLUMN meta_access_token_encrypted BYTEA;
```

Encrypt at rest (pgcrypto), decrypt per-request. Complex; do it later. For the capstone, the shared token is fine.

---

## Checkpoint

1. Sending a message to Tenant 1's WhatsApp number triggers the bot.
2. The bot replies with Tenant 1's name and products.
3. Tenant 2's bot responds differently with Tenant 2's data.
4. A message to an unknown phone_number_id logs a warning and does not crash.
5. Customer and message rows appear in the correct tenant's tables.

Commit:

```bash
git add .
git commit -m "feat: per-tenant whatsapp webhook routing and dispatcher"
```

---

## What You Learned

- Meta webhooks contain `phone_number_id` which maps to a tenant.
- One webhook URL handles many tenants.
- RLS context is set per-message inside the loop.

Tomorrow: M-Pesa per tenant with wallet crediting.
