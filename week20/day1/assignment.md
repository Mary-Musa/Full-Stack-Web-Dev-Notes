# Week 20 - Day 1 Assignment

## Title
Telegram Bot -- First Handshake With BotFather

## Overview
Week 20 adds Telegram as a fourth channel. Today you create a bot via BotFather, wire it to a webhook on your Express server, and get the first "/start" reply working from your own phone.

## Learning Objectives Assessed
- Register a Telegram bot via BotFather
- Receive the bot token and store it safely
- Set up a webhook for receiving updates
- Reply to a `/start` message from your phone

## Prerequisites
- Weeks 11 and 12 (Express + Meta webhook patterns are similar)

## AI Usage Rules

**Ratio this week:** 40% manual / 60% AI
**Habit:** New channel, manual handshake. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Repeated patterns (Express routes, env config).
- **NOT ALLOWED FOR:** The first handshake. Write it yourself.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Create a bot via BotFather

**What to do:**
1. Open Telegram.
2. Search for `@BotFather` and start a chat.
3. Send `/newbot`.
4. Follow prompts: pick a display name and a username (must end in `bot`).
5. BotFather gives you a token like `123456:ABC-DEF...`. This is your `TELEGRAM_BOT_TOKEN`.

Add to `.env`:

```
TELEGRAM_BOT_TOKEN=...
```

**Expected output:**
Bot created. Token in env. Screenshot `day1-botfather.png`.

### Task 2: Webhook route

**What to do:**
Create `routes/telegram.js`:

```javascript
const express = require("express");
const axios = require("axios");
const router = express.Router();

const TELEGRAM_API = `https://api.telegram.org/bot${process.env.TELEGRAM_BOT_TOKEN}`;

router.post("/webhook", async (req, res) => {
  console.log("Telegram update:", JSON.stringify(req.body, null, 2));

  const message = req.body.message;
  if (!message) return res.sendStatus(200);

  const chatId = message.chat.id;
  const text = message.text || "";

  if (text === "/start") {
    await axios.post(`${TELEGRAM_API}/sendMessage`, {
      chat_id: chatId,
      text: "Karibu! I am your Mctaba bot. Send /help for commands.",
    });
  }

  res.sendStatus(200);
});

module.exports = router;
```

Mount at `/telegram`. Hand-typed.

**Expected output:**
Route in place.

### Task 3: Register the webhook

**What to do:**
Telegram pulls updates via a webhook you register via their API:

```bash
curl -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://YOUR_NGROK/telegram/webhook"}'
```

Verify:

```bash
curl "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getWebhookInfo"
```

**Expected output:**
Webhook registered. Screenshot `day1-webhook.png`.

### Task 4: First reply

**What to do:**
From your phone, open a chat with your bot and send `/start`. Your server should log the update and send back the welcome message.

**Expected output:**
Reply received on your phone. Screenshot `day1-reply.png`.

### Task 5: Handshake log

**What to do:**
In `AI_AUDIT.md`, start a "Handshake log" for Telegram listing:
- BotFather creation (manual)
- Env var setup (manual)
- Webhook route (hand-typed)
- setWebhook call (manual curl)

**Expected output:**
Audit updated.

## Stretch Goals (Optional - Extra Credit)

- Add `/help` command that lists available commands.
- Log every update to a `telegram_updates` table.
- Use the Telegram Bot API library instead of raw axios.

## Submission Requirements

- **What to submit:** Repo with route, screenshots, `AI_AUDIT.md`.
- **Deadline:** End of Day 1

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Bot created via BotFather | 15 | Screenshot. |
| Webhook route by hand | 25 | Handles /start, replies via API. |
| setWebhook registered | 20 | getWebhookInfo shows your URL. |
| First reply received | 25 | Real message on phone. |
| Handshake log in audit | 15 | Every manual step listed. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Committing the bot token.** Regenerate immediately if you do. `/revoke` in BotFather.
- **Using `getUpdates` (polling) alongside `setWebhook`.** You cannot have both active. Pick one.
- **Forgetting to return 200 from the webhook.** Telegram retries on non-200.

## Resources

- Day 1 reading: [Telegram Bot Foundations.md](./Telegram%20Bot%20Foundations.md)
- Week 20 AI boundaries: [../ai.md](../ai.md)
- Telegram Bot API: https://core.telegram.org/bots/api
