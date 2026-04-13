# Week 20, Day 1: Telegram Bot Foundations

> **AI boundaries this week:** 40% manual / 60% AI. Habit: *New channel, manual handshake -- the first successful round-trip is yours; AI scaffolds what comes after.* See [ai.md](../ai.md).

By the end of today, you have a Telegram bot running locally that responds to basic commands, registered with the BotFather, and connected through a webhook. You will reuse the state-machine pattern from Week 13 USSD -- Telegram bots are functionally "USSD with images".

**Prior-week concepts you will use today:**
- State machines for multi-step flows (Week 13, Day 2)
- Express webhook handlers (Week 11, Day 1)
- The Week 12-19 CRM backend

**Estimated time:** 3 hours

---

## The Week Ahead

| Day | Focus |
|---|---|
| Day 1 (today) | BotFather, webhook setup, echo bot. |
| Day 2 | Interactive menus with inline keyboards. |
| Day 3 | Group management commands and admin tools. |
| Day 4 | Scheduled broadcasts and reminders. |
| Day 5 | Recap. |
| Weekend | A group-management bot for a real community. |

---

## Why Telegram

Telegram is the messaging channel of choice for Kenyan SaccOS, savings groups, developer communities, and increasingly WhatsApp-avoidant organisations. Four reasons:

1. **Bots are first-class.** WhatsApp's official API is restrictive (templates, opt-ins, rate limits). Telegram has none of that.
2. **Groups scale.** WhatsApp groups cap at 1,024 members. Telegram groups go to 200,000.
3. **The bot API is free and unlimited** on any account.
4. **No line-in-the-sand with Meta.** If Meta nukes your WhatsApp number tomorrow, your Telegram bot is unaffected.

For the Marathon, Telegram is the channel you reach for when the use case is "many people, one chat, bot does most of the work". Project 4 (Chama Savings) is exactly that shape.

---

## Meet BotFather

Telegram's bot registration is itself a bot. Open Telegram, search for `@BotFather`, send `/start`. Follow its menu to create a new bot:

1. `/newbot`
2. Give it a name: "Mctaba Demo Bot"
3. Give it a username ending in `bot`: `mctaba_demo_bot`
4. BotFather responds with an HTTP API token like `7234567890:AA...`. **This is your secret.** Keep it out of git.

Add to `.env`:

```env
TELEGRAM_BOT_TOKEN=7234567890:AA...
```

---

## The Telegram Bot API

Telegram exposes a simple HTTP API at `https://api.telegram.org/bot<TOKEN>/<METHOD>`. Every interaction is either:

- **An HTTP call you make** to Telegram (`sendMessage`, `editMessageText`, `sendPhoto`, etc).
- **A webhook Telegram calls** on your server (`POST /webhook` with an `Update` object).

You register your webhook URL once:

```bash
curl "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/setWebhook?url=$PUBLIC_URL/telegram/webhook"
```

Telegram verifies the URL is HTTPS and starts delivering updates. Same pattern as Meta's WhatsApp webhook.

---

## The Minimal Bot

Install `node-telegram-bot-api`:

```bash
npm install node-telegram-bot-api
```

Create `server/services/telegram.service.js`:

```javascript
const TelegramBot = require("node-telegram-bot-api");
const env = require("../config/env");

const bot = new TelegramBot(env.TELEGRAM_BOT_TOKEN, { webHook: false });

async function sendMessage(chatId, text, extra = {}) {
  return bot.sendMessage(chatId, text, extra);
}

async function processUpdate(update) {
  try {
    if (update.message) {
      await handleMessage(update.message);
    } else if (update.callback_query) {
      await handleCallbackQuery(update.callback_query);
    }
  } catch (err) {
    console.error("Telegram update error:", err);
  }
}

async function handleMessage(message) {
  const chatId = message.chat.id;
  const text = message.text || "";

  if (text === "/start") {
    await sendMessage(chatId, "Welcome! Use /help for commands.");
    return;
  }

  if (text === "/help") {
    await sendMessage(chatId,
      "Available commands:\n" +
      "/start - start the bot\n" +
      "/help - show this message\n" +
      "/echo <text> - echo back\n"
    );
    return;
  }

  if (text.startsWith("/echo ")) {
    await sendMessage(chatId, text.slice(6));
    return;
  }

  await sendMessage(chatId, `You said: ${text}`);
}

async function handleCallbackQuery(query) {
  // placeholder for Day 2
}

module.exports = { bot, sendMessage, processUpdate };
```

And the webhook route:

```javascript
// server/routes/telegram.routes.js
const express = require("express");
const router = express.Router();
const telegram = require("../services/telegram.service");

router.post("/webhook", express.json(), async (req, res) => {
  res.json({ ok: true });
  await telegram.processUpdate(req.body);
});

module.exports = router;
```

Wire in `index.js`:

```javascript
app.use("/telegram", require("./routes/telegram.routes"));
```

Start the server. ngrok your Express on a new tunnel. Register the webhook:

```bash
curl "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/setWebhook?url=https://xxx.ngrok-free.app/telegram/webhook"
```

Now open your bot in Telegram, send `/start`, and it replies.

---

## The `Update` Object Shape

Every webhook from Telegram is an `Update` object with one of several fields set. The two most common:

- **`update.message`** -- a chat message was sent. Contains `message.chat.id`, `message.from`, `message.text`, optionally `message.photo`, `message.location`, etc.
- **`update.callback_query`** -- the user clicked an inline keyboard button. Contains the button's `data` payload.

Other fields: `edited_message`, `channel_post`, `inline_query`, `chat_join_request`. Handle them as you need them.

`chat.id` is the identifier you use for every reply. Save it. For private chats it is the user id; for groups it is a negative number. You will use it for scheduled broadcasts (Day 4).

---

## Command Parsing

Telegram commands are messages that start with `/`. A common pattern:

```javascript
function parseCommand(text) {
  if (!text?.startsWith("/")) return null;
  const [cmd, ...args] = text.slice(1).split(" ");
  // Remove bot username suffix: /help@mctaba_demo_bot -> /help
  const commandName = cmd.split("@")[0];
  return { command: commandName, args };
}
```

The `@botname` suffix appears when users type a command in a group with multiple bots; it disambiguates. Strip it and match on the bare command.

---

## Saving The Chat

Create a table for known chats:

```sql
CREATE TABLE telegram_chats (
  id BIGINT PRIMARY KEY,
  type TEXT NOT NULL,
  title TEXT,
  username TEXT,
  first_joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_active_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

On every message, upsert the chat:

```javascript
await query(
  `INSERT INTO telegram_chats (id, type, title, username)
   VALUES ($1, $2, $3, $4)
   ON CONFLICT (id) DO UPDATE SET last_active_at = NOW()`,
  [message.chat.id, message.chat.type, message.chat.title, message.chat.username]
);
```

Later you broadcast to all active chats by reading this table. Group chats come in with `type: "group"` or `"supergroup"`; private chats with `type: "private"`.

---

## Privacy Mode

By default, Telegram bots in groups only see messages that **start with a command**. If you want the bot to react to every message in a group, you must disable Privacy Mode in BotFather:

1. BotFather -> `/mybots` -> pick your bot -> Bot Settings -> Group Privacy -> Turn off.

For most bots, leave it on. A bot that reacts to every message in a 500-person group is either spam or abuse.

---

## Webhook vs Polling

`node-telegram-bot-api` supports two modes:

- **Webhook** (what we use) -- Telegram calls you. Requires a public HTTPS URL. Scales well.
- **Polling** -- your code repeatedly asks Telegram for new updates. No public URL needed. Good for local dev without ngrok.

Switch by passing `polling: true` to the constructor and omitting the webhook setup. In development if you want to avoid ngrok, polling is fine. For production, always webhook.

---

## Checkpoint

1. Your bot exists in Telegram and you can open a chat with it.
2. `/start` and `/help` return the expected responses.
3. `/echo hello world` replies with "hello world".
4. Any other message returns "You said: ...".
5. `telegram_chats` table has a row for your private chat with the bot.
6. `curl "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/getWebhookInfo"` returns your ngrok URL.
7. Stopping and restarting Express does not lose the webhook registration.

Commit:

```bash
git add .
git commit -m "feat: telegram bot foundations with webhook and command parsing"
```

---

## What You Learned

- BotFather is itself a Telegram bot.
- Registration gives you a token; keep it secret.
- Webhook mode scales; polling mode is for local dev.
- Privacy mode limits what the bot sees in groups.
- The `Update` object shape is `{ message | callback_query | ... }`.

Tomorrow we add inline keyboards and turn the echo bot into a real menu-driven flow.
