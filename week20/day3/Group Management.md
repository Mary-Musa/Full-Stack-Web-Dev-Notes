# Week 20, Day 3: Group Management

By the end of today, your bot can manage group membership: welcome new members, greet them with rules, kick spammers, promote admins, and track who is allowed to run which commands. This is the half of Telegram bots most tutorials skip, and the half Project 4 actually needs.

**Prior concepts:** inline keyboards and state (Week 20 Day 2), admin auth patterns (Week 12 Day 3).

**Estimated time:** 3 hours

---

## Making Your Bot A Group Admin

For most group management features, your bot must be an administrator of the group with appropriate privileges. A user (the group owner) promotes the bot:

1. Open the group in Telegram.
2. Group info -> Administrators -> Add administrator.
3. Search for your bot username.
4. Grant permissions: Delete messages, Ban users, Add new admins (if your bot will manage other admins).

Only then can the bot's `banChatMember`, `deleteMessage`, and `promoteChatMember` calls work. Without admin status they return 403.

---

## The `chat_member` Update

When anyone joins or leaves a group the bot is in, Telegram fires a `chat_member` update. To receive it, you must explicitly subscribe:

```bash
curl "https://api.telegram.org/bot$TOKEN/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://xxx.ngrok-free.app/telegram/webhook",
    "allowed_updates": ["message", "callback_query", "chat_member", "my_chat_member"]
  }'
```

`chat_member` = someone else's membership changed.
`my_chat_member` = the bot's own membership changed (added to or removed from a group).

Now your webhook handler sees them:

```javascript
async function processUpdate(update) {
  if (update.message) return handleMessage(update.message);
  if (update.callback_query) return handleCallbackQuery(update.callback_query);
  if (update.chat_member) return handleChatMemberUpdate(update.chat_member);
  if (update.my_chat_member) return handleBotMembershipUpdate(update.my_chat_member);
}
```

---

## Welcoming New Members

```javascript
async function handleChatMemberUpdate(event) {
  const oldStatus = event.old_chat_member.status;
  const newStatus = event.new_chat_member.status;
  const user = event.new_chat_member.user;

  // A user joined
  if ((oldStatus === "left" || oldStatus === "kicked") && newStatus === "member") {
    await bot.sendMessage(
      event.chat.id,
      `Karibu ${user.first_name}! Please read the pinned message for the group rules.`
    );
    await query(
      `INSERT INTO group_members (chat_id, user_id, first_name, joined_at)
       VALUES ($1, $2, $3, NOW())
       ON CONFLICT (chat_id, user_id) DO UPDATE SET first_name = $3`,
      [event.chat.id, user.id, user.first_name]
    );
  }

  // A user left
  if ((oldStatus === "member" || oldStatus === "administrator") && newStatus === "left") {
    await query(
      `UPDATE group_members SET left_at = NOW() WHERE chat_id = $1 AND user_id = $2`,
      [event.chat.id, user.id]
    );
  }
}
```

Schema:

```sql
CREATE TABLE group_members (
  chat_id BIGINT NOT NULL,
  user_id BIGINT NOT NULL,
  first_name TEXT,
  joined_at TIMESTAMPTZ NOT NULL,
  left_at TIMESTAMPTZ,
  role TEXT DEFAULT 'member',
  PRIMARY KEY (chat_id, user_id)
);
```

You now have a local record of every member. For Project 4 this is the membership ledger.

---

## Rules Message and Accept Flow

New members should accept the rules before they can talk. Telegram has no "acceptance required" feature; you implement it with message restrictions.

```javascript
// On member join, restrict them immediately
await bot.restrictChatMember(chatId, userId, {
  permissions: { can_send_messages: false },
});

// Send a rules message with an Accept button
await bot.sendMessage(chatId,
  `Welcome ${user.first_name}! Please read and accept the rules:\n\n1. Be kind.\n2. No spam.\n3. English or Kiswahili only.`,
  {
    reply_markup: {
      inline_keyboard: [[{ text: "Accept rules", callback_data: `acc:${userId}` }]],
    },
  }
);
```

And the accept handler:

```javascript
// In handleCallbackQuery:
if (data.startsWith("acc:")) {
  const allowedUserId = parseInt(data.slice(4), 10);
  if (query.from.id !== allowedUserId) {
    return bot.answerCallbackQuery(query.id, {
      text: "This button is not for you.",
      show_alert: true,
    });
  }
  await bot.restrictChatMember(query.message.chat.id, allowedUserId, {
    permissions: { can_send_messages: true, can_send_media_messages: true, /* ... */ },
  });
  await bot.editMessageText(
    `Thanks ${query.from.first_name}! You can now participate.`,
    { chat_id: query.message.chat.id, message_id: query.message.message_id }
  );
}
```

The `allowedUserId` check is important: without it, any member in the group could click the accept button for another user.

---

## Spam Detection and Kick

Simple spam detection: users who post more than N messages per minute get kicked automatically.

```javascript
// Track message counts in Redis (faster than Postgres for hot writes)
async function incrementMessageCount(chatId, userId) {
  const client = await getClient();
  const key = `spam:${chatId}:${userId}`;
  const count = await client.incr(key);
  if (count === 1) await client.expire(key, 60); // reset every minute
  return count;
}

// In handleMessage:
const count = await incrementMessageCount(chatId, userId);
if (count > 10) {
  await bot.sendMessage(chatId, `${message.from.first_name} is posting too fast. Muted for 5 minutes.`);
  await bot.restrictChatMember(chatId, userId, {
    permissions: { can_send_messages: false },
    until_date: Math.floor(Date.now() / 1000) + 300,
  });
  return;
}
```

This is a blunt instrument. Real spam detection uses content classification, but for a savings group of 50 people, "don't post 10 messages in 60 seconds" catches most accidents and most spammers.

---

## Admin-Only Commands

Some commands should only work for group admins. Check membership status:

```javascript
async function isGroupAdmin(chatId, userId) {
  try {
    const member = await bot.getChatMember(chatId, userId);
    return member.status === "creator" || member.status === "administrator";
  } catch {
    return false;
  }
}

// In handleMessage for a protected command:
if (text === "/kick" && message.reply_to_message) {
  if (!(await isGroupAdmin(chatId, userId))) {
    return bot.sendMessage(chatId, "Only admins can kick members.");
  }
  const targetId = message.reply_to_message.from.id;
  await bot.banChatMember(chatId, targetId);
  await bot.unbanChatMember(chatId, targetId); // "kick" = ban + unban immediately
  await bot.sendMessage(chatId, "Kicked.");
}
```

`/kick` is a reply command -- the admin replies to the offending user's message with `/kick`. This is Telegram idiom for "target this user".

---

## Group-Specific Settings

Different groups may want different rules (quiet hours, language filters, contribution schedules). Store per-group settings:

```sql
CREATE TABLE group_settings (
  chat_id BIGINT PRIMARY KEY REFERENCES telegram_chats(id),
  rules_text TEXT,
  welcome_message TEXT,
  max_messages_per_minute INTEGER DEFAULT 10,
  quiet_hours_start TIME,
  quiet_hours_end TIME,
  language TEXT DEFAULT 'en'
);
```

On each message you fetch the group's settings from a small Redis cache (keyed by `chat_id`, 5 min TTL). The bot behaves differently per group without a restart.

---

## Checkpoint

1. Adding the bot to a group and promoting it to admin works.
2. A new member gets a welcome message and is temporarily restricted until they accept rules.
3. Clicking Accept grants them permission and edits the rules message to thank them.
4. Posting 11 messages in a row within 60 seconds mutes you for 5 minutes.
5. `/kick` as a reply works when issued by a group admin; fails silently (with a message) for non-admins.
6. `group_members` table has rows for everyone in your test group.
7. `group_settings` can be edited and the bot reads the new values within 5 minutes.

Commit:

```bash
git add .
git commit -m "feat: telegram group management with rules anti-spam and admin commands"
```

---

## What You Learned

- The bot must be a group admin for management features.
- `chat_member` updates are opt-in via `allowed_updates`.
- Welcome + rule-accept uses `restrictChatMember` plus inline keyboards.
- Anti-spam is a Redis counter with TTL, not a database table.
- "Kick" = ban + unban.
- Per-group settings let one bot serve many communities.

Tomorrow: broadcasts and scheduled reminders. The bot starts doing work while no one is watching.
