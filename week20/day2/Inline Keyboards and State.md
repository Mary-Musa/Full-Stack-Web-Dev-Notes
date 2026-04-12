# Week 20, Day 2: Inline Keyboards and State

By the end of today, your bot has interactive menus, holds state across messages, and looks like a real app rather than an echo toy. You will reuse the Week 13 Redis state machine pattern.

**Prior concepts:** Telegram bot foundations (Week 20 Day 1), USSD state machines (Week 13 Day 2).

**Estimated time:** 3 hours

---

## Inline Keyboards

Telegram lets you send messages with clickable buttons. The buttons can either:
- **Open a URL** -- useful for "Visit our shop" or "Pay with Stripe".
- **Send a callback** -- a hidden data string that comes back to your bot as a `callback_query`.

Example:

```javascript
await bot.sendMessage(chatId, "Welcome to Chama Bot! What would you like to do?", {
  reply_markup: {
    inline_keyboard: [
      [{ text: "Contribute", callback_data: "contribute" }],
      [{ text: "My balance", callback_data: "balance" }],
      [{ text: "Group stats", callback_data: "stats" }],
    ],
  },
});
```

The user sees three buttons. Clicking "Contribute" sends a `callback_query` back to your webhook with `data: "contribute"`. Your handler reads that, figures out what to do, and responds.

---

## Handling Callback Queries

```javascript
async function handleCallbackQuery(query) {
  const chatId = query.message.chat.id;
  const data = query.data;
  const userId = query.from.id;

  // Always acknowledge so the spinning icon goes away
  await bot.answerCallbackQuery(query.id);

  if (data === "contribute") {
    await promptContribution(chatId, userId);
  } else if (data === "balance") {
    await showBalance(chatId, userId);
  } else if (data === "stats") {
    await showStats(chatId);
  } else if (data.startsWith("amt:")) {
    const amount = parseInt(data.slice(4), 10);
    await confirmContribution(chatId, userId, amount);
  }
}
```

Always call `answerCallbackQuery(query.id)` -- without it, Telegram shows a spinner on the user's screen indefinitely.

Callback data is limited to 64 bytes. Pack it densely: short prefixes, numeric ids. A pattern like `cnf:amt:500` is better than `confirm_contribution_amount_500`.

---

## State In Redis

Telegram gives you a `chat_id` and `user_id` on every interaction. Combine them to form a Redis key and store session state just like Week 13's USSD:

```javascript
const { getClient } = require("../config/redis");
const SESSION_TTL = 600; // 10 minutes

const sessionKey = (chatId, userId) => `telegram:session:${chatId}:${userId}`;

async function getSession(chatId, userId) {
  const client = await getClient();
  const raw = await client.get(sessionKey(chatId, userId));
  return raw ? JSON.parse(raw) : { state: "idle", context: {} };
}

async function setSession(chatId, userId, data) {
  const client = await getClient();
  await client.set(sessionKey(chatId, userId), JSON.stringify(data), { EX: SESSION_TTL });
}
```

Now multi-step flows work. Example: contribute flow.

```javascript
async function promptContribution(chatId, userId) {
  await setSession(chatId, userId, { state: "awaiting_amount", context: {} });
  await bot.sendMessage(chatId, "How much are you contributing?", {
    reply_markup: {
      inline_keyboard: [
        [
          { text: "KSh 500", callback_data: "amt:500" },
          { text: "KSh 1000", callback_data: "amt:1000" },
          { text: "KSh 2000", callback_data: "amt:2000" },
        ],
        [{ text: "Custom amount", callback_data: "amt:custom" }],
      ],
    },
  });
}

async function confirmContribution(chatId, userId, amount) {
  await setSession(chatId, userId, { state: "confirming", context: { amount } });
  await bot.sendMessage(chatId, `Confirm contribution of KSh ${amount}?`, {
    reply_markup: {
      inline_keyboard: [
        [
          { text: "Yes, contribute", callback_data: "cnf:yes" },
          { text: "Cancel", callback_data: "cnf:no" },
        ],
      ],
    },
  });
}
```

---

## Free Text Handling Inside A Flow

What if the user picks "Custom amount"? They need to type a number. But Telegram's next inbound message will land in `handleMessage`, not `handleCallbackQuery`. Your message handler must check the current state and route accordingly:

```javascript
async function handleMessage(message) {
  const chatId = message.chat.id;
  const userId = message.from.id;
  const text = message.text || "";

  const session = await getSession(chatId, userId);

  if (session.state === "awaiting_custom_amount") {
    const amount = parseInt(text, 10);
    if (isNaN(amount) || amount <= 0) {
      await bot.sendMessage(chatId, "Please type a number greater than 0.");
      return;
    }
    await confirmContribution(chatId, userId, amount);
    return;
  }

  // ... normal command parsing ...
}
```

And when the user clicks "Custom amount":

```javascript
if (data === "amt:custom") {
  await setSession(chatId, userId, { state: "awaiting_custom_amount", context: {} });
  await bot.sendMessage(chatId, "Type the amount in KSh:");
}
```

This is the bridge between callback-driven and text-driven flows. The Redis state machine keeps them in sync.

---

## Editing Messages

A key feature of Telegram inline keyboards: you can edit an existing message instead of sending a new one. This keeps the chat clean.

```javascript
await bot.editMessageText(`Confirming KSh ${amount}...`, {
  chat_id: chatId,
  message_id: query.message.message_id,
  reply_markup: {
    inline_keyboard: [[{ text: "Processing...", callback_data: "noop" }]],
  },
});
```

The original message updates in place. Good UX: after a button click, remove the buttons or replace them with a processing state so the user cannot double-click.

---

## Markdown and HTML Formatting

Telegram supports two parse modes: Markdown and HTML.

```javascript
await bot.sendMessage(chatId, "*Bold* and _italic_ and [a link](https://example.com)", {
  parse_mode: "Markdown",
});
```

Markdown is fine for simple styling. HTML is safer for user-generated content because you have explicit escaping rules:

```javascript
function escapeHtml(text) {
  return text.replace(/[&<>]/g, (c) => ({ "&": "&amp;", "<": "&lt;", ">": "&gt;" }[c]));
}

await bot.sendMessage(chatId, `<b>${escapeHtml(userName)}</b> joined`, { parse_mode: "HTML" });
```

Never send unescaped user input with `parse_mode: "Markdown"` -- a user with a name containing `*`, `_`, or `[` will break your message.

---

## Checkpoint

1. Clicking Contribute shows three preset amounts plus "Custom".
2. Clicking an amount shows a Yes/Cancel confirmation.
3. Clicking Yes writes a contribution row to Postgres and sends a success message.
4. Clicking Custom and typing a number continues the flow.
5. Typing random text in the middle of a flow does not crash -- re-prompt.
6. Two users in the same group chat can have independent sessions (because the Redis key includes `user_id`).

Commit:

```bash
git add .
git commit -m "feat: inline keyboards and redis-backed session state"
```

---

## What You Learned

- Inline keyboards send `callback_query` events with a 64-byte `data` string.
- `answerCallbackQuery` must be called or the spinner hangs.
- Redis holds session state keyed by chat+user.
- Free-text prompts and callback flows share the same state machine.
- `editMessageText` updates the original message instead of adding clutter.

Tomorrow we handle group management: who can join, who can speak, who can run commands, and how to moderate.
