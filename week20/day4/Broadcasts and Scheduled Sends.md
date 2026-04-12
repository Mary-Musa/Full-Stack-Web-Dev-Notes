# Week 20, Day 4: Broadcasts and Scheduled Sends

By the end of today, your bot can broadcast a message to every group it is in, send scheduled reminders, and rate-limit itself so Telegram does not throttle it. You will install `node-cron` as a preview of Week 21's material.

**Prior concepts:** group tracking (Week 20 Day 3), Redis for counters (Week 20 Day 3).

**Estimated time:** 3 hours

---

## Broadcasting A Message

"Send this to every group the bot is in." The naive version:

```javascript
async function broadcastNaive(text) {
  const { rows } = await query("SELECT id FROM telegram_chats WHERE type IN ('group', 'supergroup')");
  for (const chat of rows) {
    await bot.sendMessage(chat.id, text);
  }
}
```

This works for ten groups. It breaks at a hundred because Telegram rate-limits bots to about 30 messages per second globally (and 20 per minute per group). Your loop sends as fast as your event loop will let it, and Telegram starts returning `429 Too Many Requests`.

The right version paces itself:

```javascript
async function broadcast(text, { rateMs = 50 } = {}) {
  const { rows } = await query("SELECT id FROM telegram_chats WHERE type IN ('group', 'supergroup')");
  let sent = 0;
  let failed = 0;

  for (const chat of rows) {
    try {
      await bot.sendMessage(chat.id, text, { parse_mode: "HTML" });
      sent++;
    } catch (err) {
      if (err.response?.body?.error_code === 403) {
        // Bot was removed from the chat; delete the record
        await query("DELETE FROM telegram_chats WHERE id = $1", [chat.id]);
      } else if (err.response?.body?.error_code === 429) {
        // Rate limited; sleep a second and retry
        const retryAfter = (err.response.body.parameters?.retry_after || 1) * 1000;
        await new Promise((r) => setTimeout(r, retryAfter));
        try {
          await bot.sendMessage(chat.id, text);
          sent++;
        } catch {
          failed++;
        }
      } else {
        failed++;
      }
    }
    await new Promise((r) => setTimeout(r, rateMs));
  }

  return { sent, failed };
}
```

Three refinements.

**Pace at 50ms between sends.** That is 20 messages/second, comfortably under Telegram's global limit.

**Handle 403 (forbidden).** If the bot has been kicked, the chat row is stale. Delete it so we do not waste future sends on dead chats.

**Handle 429 with backoff.** When Telegram tells you to slow down, it gives you a `retry_after` value. Respect it. Naive retries after 429 are what gets bots banned.

---

## Idempotent Broadcasts

If your broadcast script crashes midway, you want to resume from where it stopped, not re-broadcast to the first half. Add a broadcast record:

```sql
CREATE TABLE broadcasts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  text TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at TIMESTAMPTZ
);

CREATE TABLE broadcast_deliveries (
  broadcast_id UUID NOT NULL REFERENCES broadcasts(id),
  chat_id BIGINT NOT NULL,
  sent_at TIMESTAMPTZ,
  status TEXT,
  PRIMARY KEY (broadcast_id, chat_id)
);
```

Before each send, check if you already sent to this chat for this broadcast. If yes, skip. If the script crashes and restarts, it picks up where it left off.

This is the same "deliver exactly once" pattern as the payments outbox from Week 18. Telegram bots benefit from the same discipline.

---

## Scheduled Reminders

`node-cron` runs functions on a schedule. Install it:

```bash
npm install node-cron
```

Example: remind every chama group daily at 8am KE time to log their contributions.

```javascript
const cron = require("node-cron");

cron.schedule("0 8 * * *", async () => {
  console.log("Running daily chama reminder");
  const { rows } = await query(
    `SELECT chat_id FROM group_settings WHERE settings->>'daily_reminder' = 'true'`
  );

  for (const row of rows) {
    try {
      await bot.sendMessage(row.chat_id,
        "Good morning! Reminder: log today's contribution with /contribute"
      );
    } catch (err) {
      console.error("Reminder send failed:", err);
    }
    await new Promise((r) => setTimeout(r, 50));
  }
}, { timezone: "Africa/Nairobi" });
```

The cron spec `0 8 * * *` is "at 08:00 every day". The `timezone` option makes it East African time rather than UTC.

Week 21 we do cron properly with error handling, retry on failure, and scheduled task tracking. For today, the bare minimum works.

---

## Per-User Reminders

"Send a reminder to user X when they have not contributed in 7 days." This is more personalised than a broadcast:

```javascript
cron.schedule("0 18 * * *", async () => {
  const { rows } = await query(
    `SELECT gm.chat_id, gm.user_id, gm.first_name, MAX(c.created_at) AS last_contribution
     FROM group_members gm
     LEFT JOIN contributions c ON c.user_id = gm.user_id AND c.chat_id = gm.chat_id
     WHERE gm.left_at IS NULL
     GROUP BY gm.chat_id, gm.user_id, gm.first_name
     HAVING MAX(c.created_at) < NOW() - INTERVAL '7 days' OR MAX(c.created_at) IS NULL`
  );

  for (const row of rows) {
    try {
      // Try sending a private message first
      await bot.sendMessage(row.user_id,
        `Hi ${row.first_name}, friendly reminder that your chama contribution is overdue.`
      );
    } catch (err) {
      if (err.response?.body?.error_code === 403) {
        // User has not started a private chat with the bot
        // Fall back to @mentioning in the group
        await bot.sendMessage(row.chat_id,
          `<a href="tg://user?id=${row.user_id}">${row.first_name}</a>, your contribution is overdue.`,
          { parse_mode: "HTML" }
        );
      }
    }
    await new Promise((r) => setTimeout(r, 100));
  }
}, { timezone: "Africa/Nairobi" });
```

Two subtleties.

**Private messages require the user to have started a chat with the bot.** If they have not, `sendMessage` returns 403. The fallback is mentioning them in the group with `tg://user?id=X`, which notifies them on most Telegram clients.

**The "@mention" format** uses HTML parse mode. The `tg://user?id=NUMBER` URL is a Telegram deep link to the user's profile.

---

## A Simple Admin Broadcast Form

Add an admin page to trigger broadcasts from the web dashboard:

```jsx
// app/admin/telegram-broadcast/page.js
import { requireAdmin } from "@/app/actions/auth";
import BroadcastForm from "./BroadcastForm";

export default async function BroadcastPage() {
  await requireAdmin();
  return (
    <div className="max-w-xl mx-auto p-8">
      <h1 className="text-2xl font-bold mb-6">Telegram Broadcast</h1>
      <BroadcastForm />
    </div>
  );
}
```

`BroadcastForm` is a Client Component with a textarea and a Submit button that calls a Server Action:

```javascript
"use server";
export async function sendBroadcast(prevState, formData) {
  await requireAdmin();
  const text = formData.get("text")?.toString();
  if (!text || text.length < 3) return { error: "Message too short" };

  const res = await fetch(`${process.env.CRM_SERVER_URL}/api/telegram/broadcast`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ text }),
  });
  const data = await res.json();
  return { ok: true, sent: data.sent, failed: data.failed };
}
```

Safety features:
- Admin-only route.
- Confirmation step (type "SEND" to confirm) before dispatch.
- Rate-limit button clicks (one broadcast per 60 seconds).
- Log the broadcast to `broadcasts` table with the admin id.

Broadcasts are dangerous. One wrong click sends a spam message to every group. Build the guardrails before you need them.

---

## Checkpoint

1. A `/broadcast` admin command sends a message to every group the bot is in.
2. 50ms delay between sends avoids the Telegram rate limit.
3. Removing the bot from a group, then running a broadcast, deletes the stale chat from the table.
4. The cron job runs at 8am KE time (test by setting `*/1 * * * *` and verifying).
5. A 7-day-overdue reminder sends private messages where possible, falls back to group mentions.
6. `broadcasts` + `broadcast_deliveries` tables track every send for auditing.
7. The admin dashboard has a Broadcast page with a confirmation flow.

Commit:

```bash
git add .
git commit -m "feat: rate-limited broadcasts and scheduled telegram reminders"
```

---

## What You Learned

- Broadcasting naively hits rate limits; pace at 20 msg/sec.
- 403 errors mean the bot was removed; clean up stale records.
- 429 errors come with a `retry_after` value; respect it.
- Idempotent broadcasts survive crashes via a delivery table.
- `node-cron` with `timezone` gives you scheduled tasks in East African time.
- Private reminders fall back to group mentions when the user has not DMd the bot.

Tomorrow is the recap and the weekend project builds a group-management bot for a real community.
