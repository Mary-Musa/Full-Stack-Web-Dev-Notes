# Week 16, Day 3: WhatsApp Order Confirmations

By the end of today, every successful order sends a formatted WhatsApp message to the customer -- "Thanks Wanjiru, order #AB12CD is confirmed, total KSh 2,499, delivery by Friday" -- complete with the items, a tracking link, and a one-tap button to message support. The message fires from the same M-Pesa callback that marks the order `paid`.

You are reusing the Week 11 WhatsApp sender, just pointing it at a new purpose.

**Prior-week concepts you will use today:**
- The Week 11 WhatsApp sender (`sendWhatsAppMessage` or equivalent)
- The M-Pesa callback handler with idempotency (Week 16, Day 2)
- The `leads` + `conversations` + `messages` schema (Week 11)

**Estimated time:** 3 hours

---

## The Pattern We Are Following

WhatsApp Cloud API has two kinds of outgoing messages:

1. **Session messages** -- plain text you send *inside* a 24-hour conversation that was started by the customer. Free, unlimited, any content.
2. **Template messages** -- pre-approved message templates you send *outside* a 24-hour window. Must match an approved template, must be sent to a user who has opted in, charged per message.

For a new customer who just paid, there is no active 24-hour window (they never messaged you first). So order confirmations must be template messages.

Meta template approval is a process: you submit the text with placeholders, Meta reviews it (usually within a day), you get an ID you reference in API calls. You cannot skip this step on live Meta numbers. **On the sandbox**, you can send to numbers that have explicitly opted in (they dial `join <phrase>` to your test number first) and then send plain text within the 24-hour window without template approval.

We will build both paths:

1. **Template message** (production-correct): the function that sends an approved template with the order details.
2. **Session message** (sandbox-friendly): a fallback for development that sends plain text inside an active session.

Pick which one to call based on whether you have an approved template.

---

## Creating the Template in Meta

On https://business.facebook.com, navigate to your WhatsApp Business Account -> Message Templates -> Create Template.

Choose category: **Utility** (for transactional order confirmations). Do not pick Marketing -- Meta rejects marketing templates for order confirmations and they cost more.

Name it `order_confirmation`. Pick a language (English). Body text:

```
Asante {{1}}! Your order {{2}} has been confirmed.

Total: KSh {{3}}
Items: {{4}}

We will deliver by {{5}}. Reply to this message if you have any questions.
```

Variables `{{1}}` through `{{5}}` are placeholders you fill in at send time. Submit for approval. Usually takes 1-24 hours.

While waiting, write the sending code with the template name hardcoded. On the sandbox, you can test immediately using session messages.

---

## The Sender Module

In the Express server, create `server/services/whatsapp.service.js` (or extend the existing one from Week 11):

```javascript
// server/services/whatsapp.service.js
const axios = require("axios");
const env = require("../config/env");

const BASE_URL = "https://graph.facebook.com/v21.0";

async function sendTemplateMessage({ to, templateName, language = "en", components = [] }) {
  const res = await axios.post(
    `${BASE_URL}/${env.META_PHONE_NUMBER_ID}/messages`,
    {
      messaging_product: "whatsapp",
      to: formatPhone(to),
      type: "template",
      template: {
        name: templateName,
        language: { code: language },
        components,
      },
    },
    { headers: { Authorization: `Bearer ${env.META_ACCESS_TOKEN}` } }
  );
  return res.data;
}

async function sendTextMessage({ to, text }) {
  const res = await axios.post(
    `${BASE_URL}/${env.META_PHONE_NUMBER_ID}/messages`,
    {
      messaging_product: "whatsapp",
      to: formatPhone(to),
      type: "text",
      text: { body: text },
    },
    { headers: { Authorization: `Bearer ${env.META_ACCESS_TOKEN}` } }
  );
  return res.data;
}

function formatPhone(phone) {
  return phone.replace(/^\+/, "").replace(/\s/g, "");
}

module.exports = { sendTemplateMessage, sendTextMessage };
```

Two functions: one for templates (production path), one for plain text (sandbox / opted-in users). Both wrap the Graph API with the right headers.

---

## The Order Confirmation Function

Create `server/services/orderNotifications.service.js`:

```javascript
// server/services/orderNotifications.service.js
const { query } = require("../config/db");
const whatsapp = require("./whatsapp.service");
const env = require("../config/env");

async function sendOrderConfirmation(orderId) {
  const { rows: orderRows } = await query(
    `SELECT id, customer_name, customer_phone, subtotal_cents, status
     FROM orders WHERE id = $1`,
    [orderId]
  );
  const order = orderRows[0];
  if (!order) throw new Error("Order not found");

  const { rows: items } = await query(
    `SELECT oi.quantity, p.name
     FROM order_items oi JOIN products p ON p.id = oi.product_id
     WHERE oi.order_id = $1`,
    [orderId]
  );

  const itemsText = items.map((i) => `${i.quantity}x ${i.name}`).join(", ");
  const totalKsh = (order.subtotal_cents / 100).toLocaleString();
  const firstName = order.customer_name.split(" ")[0];
  const shortId = order.id.slice(0, 8).toUpperCase();
  const deliveryDate = computeDeliveryDate();

  if (env.USE_WHATSAPP_TEMPLATE === "true") {
    return whatsapp.sendTemplateMessage({
      to: order.customer_phone,
      templateName: "order_confirmation",
      components: [
        {
          type: "body",
          parameters: [
            { type: "text", text: firstName },
            { type: "text", text: shortId },
            { type: "text", text: totalKsh },
            { type: "text", text: itemsText },
            { type: "text", text: deliveryDate },
          ],
        },
      ],
    });
  } else {
    // Sandbox / development path
    const text = `Asante ${firstName}! Your order ${shortId} has been confirmed.

Total: KSh ${totalKsh}
Items: ${itemsText}

We will deliver by ${deliveryDate}. Reply to this message if you have any questions.`;
    return whatsapp.sendTextMessage({ to: order.customer_phone, text });
  }
}

function computeDeliveryDate() {
  const now = new Date();
  const delivery = new Date(now.getTime() + 2 * 24 * 60 * 60 * 1000);
  return delivery.toLocaleDateString("en-KE", { weekday: "long", month: "short", day: "numeric" });
}

module.exports = { sendOrderConfirmation };
```

The function reads the order and its items from Postgres, builds the message payload with the five variables, and calls the appropriate sender. One flag (`USE_WHATSAPP_TEMPLATE`) switches between template mode (production) and text mode (dev).

Now update the payments callback to call this:

```javascript
// server/services/payments.service.js
const orderNotifications = require("./orderNotifications.service");

// inside handleCallback, after marking paid:
orderNotifications.sendOrderConfirmation(order.id).catch((err) =>
  console.error("Order confirmation send failed:", err)
);
```

Fire and forget -- the caller (the callback handler) does not wait for the WhatsApp send.

---

## Logging the Outbound Message

You want a record of every message you send. Extend the Week 11 `messages` table to include outbound messages tied to orders:

```sql
ALTER TABLE messages ADD COLUMN order_id UUID REFERENCES orders(id);
CREATE INDEX idx_messages_order_id ON messages(order_id);
```

Then in `sendOrderConfirmation`, after the `axios` call succeeds:

```javascript
await query(
  `INSERT INTO messages (lead_id, direction, body, order_id)
   VALUES (NULL, 'out', $1, $2)`,
  [itemsText, orderId]
);
```

Two uses for this log:

1. **Debugging.** When a customer complains "I never got my confirmation", you can check the messages table and prove you sent it.
2. **Admin view.** On the admin order detail page, show the "Messages" section listing every outbound message attached to the order. Copy the component from Week 11 Day 4.

---

## Test It On The Sandbox

Meta's sandbox has one quirk: you can only send messages to numbers that have first messaged *your* test number. So:

1. Save your test number to your contacts.
2. Send any message to it from your phone.
3. Now your phone is "opted in" for 24 hours.
4. Place an order on the shop using *your* phone number.
5. Complete the M-Pesa sandbox payment.
6. Within a second or two, your real WhatsApp receives the confirmation message.

The first time this works is the third goose-bump moment of the Marathon (M-Pesa STK prompt was the second). You built a system where someone buys a thing and gets a message on their phone. Real commerce.

### When things go wrong

- **401 Unauthorized** from Meta -- your access token is wrong or expired. Sandbox tokens rotate every 24 hours; regenerate and update `.env`.
- **"Recipient is not in allowed list"** -- you have not opted in; send a message from your phone to the test number first.
- **"Message failed: templates disabled"** -- you tried to send a template that is not approved yet. Switch to `USE_WHATSAPP_TEMPLATE=false` during development.
- **No message received** but API says success -- check WhatsApp spam/archived chats. Sandbox messages land there sometimes.

---

## The "Reply to this message" Flow

The confirmation says "Reply to this message if you have any questions". What happens when the customer actually does?

The customer's reply hits your Week 11 webhook. Your bot state machine would pick it up as a new lead and put them through the onboarding flow -- which is wrong. You want order-related replies to go to a support queue, not restart the lead bot.

The fix: when a reply comes in with `in_reply_to` pointing at one of your messages, look up which order that message was about, attach the reply to that order, and skip the lead bot entirely.

The Graph API delivers `context.id` on any reply that points at the original message id. Store the message id from the Meta send response:

```javascript
const sendResult = await whatsapp.sendTemplateMessage(...);
const metaMessageId = sendResult.messages?.[0]?.id;

await query(
  `INSERT INTO messages (lead_id, direction, body, order_id, meta_message_id)
   VALUES (NULL, 'out', $1, $2, $3)`,
  [itemsText, orderId, metaMessageId]
);
```

Add a `meta_message_id TEXT` column to messages. Then in the webhook handler:

```javascript
const context = incoming?.context?.id;
if (context) {
  const { rows } = await query(
    "SELECT order_id FROM messages WHERE meta_message_id = $1",
    [context]
  );
  const orderId = rows[0]?.order_id;
  if (orderId) {
    // Attach as a reply to the order, not a new lead
    await query(
      `INSERT INTO messages (lead_id, direction, body, order_id)
       VALUES (NULL, 'in', $1, $2)`,
      [incoming.text.body, orderId]
    );
    return; // skip the bot
  }
}
```

Now order replies end up in `/admin/orders/[id]` under a "Messages" tab, not in the CRM leads pipeline. Clean separation.

---

## A Short Note On Rate Limits

Meta rate-limits outbound template messages. The limits scale with your account "tier":

- Tier 1 (sandbox / new): 250 messages per 24 hours.
- Tier 2: 1,000 per 24 hours.
- Tier 3: 10,000 per 24 hours.

You graduate tiers by having a good delivery rate and opted-in recipients. For a small Nairobi e-commerce shop you probably never hit the limit, but an evening rush of 50 orders plus 50 delivery updates is already 100 template messages.

Two mitigations:

1. **Queue sends in Redis / BullMQ** (Week 23 builds this properly) so you can retry with backoff when Meta says "slow down".
2. **Batch low-priority sends** (daily stock alerts, marketing) into the quiet hours so real-time transactional sends have the budget.

For Project 3 you do not need to solve this; just be aware of it.

---

## Checkpoint

1. Placing an order and completing payment sends a real WhatsApp message to the buyer (in sandbox, after opting in).
2. The message contains the customer's first name, order id, total, items, and delivery date.
3. `messages` table has a new row with `direction = 'out'` and `order_id` set.
4. Replying to the confirmation from WhatsApp creates a new `messages` row with the same `order_id`, not a new lead.
5. Admin order detail page has a Messages section showing both the outbound confirmation and any inbound replies.
6. Failing the Meta API call (bad token, disabled template) does not crash the payments callback -- it logs and continues.

Commit:

```bash
git add .
git commit -m "feat: whatsapp order confirmations with reply threading"
```

---

## What You Learned

- WhatsApp has two paths: templates (production, approved) and session text (dev, opted-in).
- Order confirmations must be utility templates, not marketing.
- Fire-and-forget from the payments callback; never block payment processing on notifications.
- Log every outbound message for debugging and admin visibility.
- Use `context.id` to thread replies back to the right order, not into the lead bot.

Tomorrow we polish Project 3: admin notifications, failed-payment retries, and the final assembly. Friday is the recap, and the weekend ships Project 3 end-to-end.
