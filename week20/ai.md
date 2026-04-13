# Week 20 - AI Boundaries

**Ratio this week: 40% Manual / 60% AI**
**Habit introduced: "New channel, manual handshake."**
**Shift from last week: Small dip back up because Telegram is a new API for you.**

This week you add Telegram bots to your toolkit. The Telegram Bot API is different in shape from WhatsApp Cloud API -- fewer restrictions, webhook and polling modes, inline keyboards, groups. The ratio dips slightly because Telegram is new, then will rebound next week when cron becomes a pattern you already know in a new wrapper.

The habit applies to every new communication channel: the *handshake* -- the initial setup, auth, and first successful message round-trip -- is done manually. Once you can send and receive your first message, AI can scaffold menus, commands, and flows.

---

## What You MUST Do Manually (40%)

### Day 1 -- Telegram bot foundations
- Create a bot with BotFather yourself. Get the token. Store it in `.env`.
- Read the Telegram Bot API docs. Know the two modes: webhook vs polling.
- Write the first /start handler by hand. Make the bot reply with "Hello, $name".
- Expose via ngrok. Register the webhook.

### Day 2 -- Inline keyboards and state
- Inline keyboards are Telegram's equivalent of buttons. Design one keyboard on paper first (think Week 15 state machine habit).
- Write the callback query handler by hand. Understand how clicks return to your server.

### Day 3 -- Group management
- Add the bot to a real group (yours, a test one). Understand what permissions it has.
- Handle commands in groups vs private chats. The `/command@botname` format for groups.
- Implement `/kick`, `/mute`, or `/rules` -- something a real group would want.

### Day 4 -- Broadcasts and scheduled sends
- Send a broadcast to multiple users. Rate-limit manually to respect Telegram's limits.
- Schedule a message for later. This previews Week 21's cron work.

---

## What You CAN Use AI For (60%)

- **Menu text and wording.**
- **Command handler boilerplate** after you wrote the first few.
- **Database schema for subscribers, groups, etc.** -- repeated pattern.
- **Rate limiter wrapper** after you understand Telegram's limits.

Forbidden:
- First handshake (webhook verify + first reply).
- Token storage.
- Anything touching the Bot API token.

### Good vs bad prompts this week

**Bad:** "Build a Telegram bot for me."
**Good:** "Here is my /start handler [paste]. I want /help to list all my commands with a one-line description each. What is the simplest way to define this once and reuse it?"

---

## Things AI Is Bad At This Week

- **Version of the Bot API.** Telegram updates its API; AI's training may be old. Check the official changelog.
- **Rate limits.** Telegram's limits are complex (global, per-chat, per-user). AI often gets the numbers wrong. Read the docs.
- **Inline vs reply keyboards.** AI mixes them up. They behave very differently.

---

## Core Mental Models For This Week

- **A bot is a user that runs on a server.** Same API, different intent.
- **An update is the unit of inbound work.** You handle one at a time; Telegram guarantees order.
- **An inline keyboard is a form.** Buttons send `callback_query` events your server handles.

---

## This Week's AI Audit Focus

Add a **handshake log**: the exact sequence from BotFather to first reply, with every manual step listed.

---

## Assessment

- Live test: send /start to your bot from a real phone. Facilitator responds via Telegram with follow-up commands.
- Explain webhook vs polling trade-offs.
- Walk through one callback query handler.

---

## One Sentence To Remember

"Every new channel is a new handshake. Do the handshake yourself; let AI scaffold what comes after."
