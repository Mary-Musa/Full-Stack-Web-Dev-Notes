# Week 29, Day 4: Unified Inbox and Dashboard

By the end of today, the tenant's dashboard shows a single inbox with all incoming messages and orders across USSD, WhatsApp, and web. One place to work, three channels feeding it.

**Estimated time:** 3-4 hours.

---

## The Unified Inbox

All messages land in the `messages` table with a `channel` column. The inbox is a single query across channels, grouped by customer.

```jsx
// shop/app/dashboard/inbox/page.js
import { queryAsTenant } from "@/lib/db";
import { requireAuth } from "@/app/lib/auth";
import Link from "next/link";

export default async function InboxPage({ searchParams }) {
  const user = await requireAuth();
  const channel = searchParams.channel;

  const conditions = ["m.tenant_id = $1"];
  const params = [user.tenantId];
  if (channel) {
    params.push(channel);
    conditions.push(`m.channel = $${params.length}`);
  }

  // Latest message per customer
  const { rows: conversations } = await queryAsTenant(user.tenantId,
    `SELECT DISTINCT ON (m.customer_id)
       m.customer_id, c.phone, c.name, m.channel, m.direction, m.body, m.created_at
     FROM messages m
     LEFT JOIN customers c ON c.id = m.customer_id
     WHERE ${conditions.join(" AND ")}
     ORDER BY m.customer_id, m.created_at DESC
     LIMIT 100`,
    params
  );

  return (
    <div className="p-8">
      <h1 className="text-2xl font-bold mb-6">Inbox</h1>

      <nav className="flex gap-3 mb-6 text-sm">
        {["", "whatsapp", "ussd", "web"].map((c) => (
          <Link
            key={c || "all"}
            href={c ? `/dashboard/inbox?channel=${c}` : "/dashboard/inbox"}
            className={`px-3 py-1 rounded ${channel === c ? "bg-black text-white" : "bg-gray-100"}`}
          >
            {c || "all"}
          </Link>
        ))}
      </nav>

      <ul className="divide-y border rounded">
        {conversations.map((c) => (
          <li key={c.customer_id}>
            <Link href={`/dashboard/inbox/${c.customer_id}`} className="block p-4 hover:bg-gray-50">
              <div className="flex justify-between">
                <div className="font-medium">{c.name || c.phone}</div>
                <div className="text-xs text-gray-500">
                  {c.channel} - {new Date(c.created_at).toLocaleString("en-KE")}
                </div>
              </div>
              <div className="text-sm text-gray-700 mt-1">
                {c.direction === "in" ? "" : "you: "}
                {c.body?.slice(0, 100)}
              </div>
            </Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

`DISTINCT ON` picks the most recent message per customer. One row per conversation, not one per message.

---

## Conversation Detail

```jsx
// shop/app/dashboard/inbox/[customerId]/page.js
import { queryAsTenant } from "@/lib/db";
import { requireAuth } from "@/app/lib/auth";

export default async function ConversationPage({ params }) {
  const user = await requireAuth();

  const { rows: customer } = await queryAsTenant(user.tenantId,
    `SELECT * FROM customers WHERE id = $1`, [params.customerId]
  );

  const { rows: messages } = await queryAsTenant(user.tenantId,
    `SELECT * FROM messages WHERE customer_id = $1 ORDER BY created_at ASC LIMIT 200`,
    [params.customerId]
  );

  const { rows: orders } = await queryAsTenant(user.tenantId,
    `SELECT * FROM orders WHERE customer_id = $1 ORDER BY created_at DESC LIMIT 10`,
    [params.customerId]
  );

  return (
    <div className="p-8 grid grid-cols-3 gap-6">
      <div className="col-span-2">
        <h1 className="text-2xl font-bold mb-4">
          {customer[0]?.name || customer[0]?.phone}
        </h1>

        <div className="space-y-3">
          {messages.map((m) => (
            <div key={m.id} className={`p-3 rounded ${m.direction === "in" ? "bg-gray-100" : "bg-blue-100 ml-12"}`}>
              <div className="text-xs text-gray-500 mb-1">
                {m.channel} - {new Date(m.created_at).toLocaleString("en-KE")}
              </div>
              <div>{m.body}</div>
            </div>
          ))}
        </div>
      </div>

      <aside>
        <h2 className="font-semibold mb-2">Customer info</h2>
        <p className="text-sm">{customer[0]?.phone}</p>
        <p className="text-xs text-gray-500 mb-4">
          First seen: {new Date(customer[0]?.first_seen_at).toLocaleDateString("en-KE")}
        </p>

        <h2 className="font-semibold mb-2 mt-6">Recent orders</h2>
        <ul className="space-y-2 text-sm">
          {orders.map((o) => (
            <li key={o.id} className="border rounded p-2">
              <div className="font-mono text-xs">{o.id.slice(0, 8)}</div>
              <div>KSh {(o.total_cents / 100).toLocaleString()} - {o.status}</div>
            </li>
          ))}
        </ul>
      </aside>
    </div>
  );
}
```

Conversation pane + customer sidebar + recent orders. One screen shows everything a sales rep needs.

---

## Replying From The Dashboard

Add a reply form at the bottom of the conversation view:

```jsx
"use client";
import { replyToCustomer } from "@/app/actions/messages";

export default function ReplyForm({ customerId }) {
  const [state, formAction] = useActionState(replyToCustomer, { error: null });
  return (
    <form action={formAction} className="mt-6 flex gap-2">
      <input type="hidden" name="customerId" value={customerId} />
      <textarea name="text" required className="flex-1 border p-2" placeholder="Type a reply..." />
      <button type="submit" className="bg-black text-white px-4">Send</button>
    </form>
  );
}
```

```javascript
// shop/app/actions/messages.js
"use server";
import { requireAuth } from "@/app/lib/auth";
import { queryAsTenant } from "@/lib/db";

export async function replyToCustomer(prevState, formData) {
  const user = await requireAuth();
  const customerId = formData.get("customerId")?.toString();
  const text = formData.get("text")?.toString();
  if (!text || text.length < 1) return { error: "Empty message" };

  // Find the customer's last channel
  const { rows } = await queryAsTenant(user.tenantId,
    `SELECT channel, phone FROM messages m JOIN customers c ON c.id = m.customer_id
     WHERE m.customer_id = $1 ORDER BY m.created_at DESC LIMIT 1`,
    [customerId]
  );
  const last = rows[0];
  if (!last) return { error: "No previous message" };

  // Dispatch via the right channel
  if (last.channel === "whatsapp") {
    await fetch(`${process.env.CRM_SERVER_URL}/internal/whatsapp/send`, {
      method: "POST",
      headers: { "Content-Type": "application/json", "X-Internal": process.env.INTERNAL_SECRET },
      body: JSON.stringify({ tenantId: user.tenantId, to: last.phone, text }),
    });
  } else if (last.channel === "ussd") {
    // USSD is session-bound; replies go via SMS as fallback
    await fetch(`${process.env.CRM_SERVER_URL}/internal/sms/send`, { /* ... */ });
  }

  // Log in the messages table
  await queryAsTenant(user.tenantId,
    `INSERT INTO messages (tenant_id, customer_id, channel, direction, body) VALUES ($1, $2, $3, 'out', $4)`,
    [user.tenantId, customerId, last.channel, text]
  );

  return { ok: true };
}
```

Dashboard users reply. The reply routes back through the same channel the customer used -- WhatsApp reply for WhatsApp messages, SMS for USSD.

---

## The Overview Page Refreshed

Update the dashboard overview to show multi-channel stats:

```sql
SELECT
  (SELECT COUNT(*) FROM orders WHERE tenant_id = $1 AND created_at::date = CURRENT_DATE) AS orders_today,
  (SELECT COUNT(*) FROM orders WHERE tenant_id = $1 AND channel = 'ussd' AND created_at::date = CURRENT_DATE) AS orders_ussd_today,
  (SELECT COUNT(*) FROM orders WHERE tenant_id = $1 AND channel = 'whatsapp' AND created_at::date = CURRENT_DATE) AS orders_whatsapp_today,
  (SELECT COUNT(*) FROM messages WHERE tenant_id = $1 AND direction = 'in' AND created_at::date = CURRENT_DATE) AS messages_today,
  (SELECT balance_cents FROM wallets WHERE tenant_id = $1) AS wallet_balance
```

Seven widgets. Owner sees at a glance: "5 orders today, 2 from USSD, 3 from WhatsApp, 12 messages, wallet KSh 15,000".

---

## Checkpoint

1. Inbox shows one row per customer with the latest message.
2. Filter by channel works (USSD, WhatsApp, all).
3. Clicking a conversation shows the full message thread plus customer info and orders.
4. Replying sends through the right channel and logs in messages.
5. Overview page shows multi-channel stats.

Commit:

```bash
git add .
git commit -m "feat: unified inbox and multi-channel dashboard"
```

---

## What You Learned

- `DISTINCT ON` picks one row per group in one query.
- Conversations are message threads linked by customer id.
- Replies route back through the originating channel.
- The overview page is the owner's favourite screen.

Tomorrow: full demo of the SME OS.
