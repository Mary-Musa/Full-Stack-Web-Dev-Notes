# Week 20, Day 5: Week Recap

Telegram is your fourth communication channel after WhatsApp, web dashboard, and USSD. Project 4 (Chama Savings) next week builds on everything this week shipped.

---

## What You Built

1. Telegram bot registered with BotFather, connected via webhook.
2. Command parser handling `/start`, `/help`, `/echo`, plus custom commands.
3. Inline keyboards with callback queries.
4. Redis-backed session state for multi-step flows.
5. Group member tracking via `chat_member` updates.
6. Welcome flow with temporary restrictions and rule acceptance.
7. Anti-spam via Redis counters with TTL.
8. Admin-only commands (`/kick`, `/promote`) gated on group admin status.
9. Per-group settings in Postgres.
10. Rate-limited broadcasts with 429 backoff.
11. Idempotent broadcast delivery tracking.
12. `node-cron` scheduled reminders with Africa/Nairobi timezone.

---

## Self-Review Questions

1. What is the difference between `update.message` and `update.callback_query`?
2. Why do you call `answerCallbackQuery`?
3. What is the 64-byte limit on inline keyboard callback data, and how do you work around it?
4. What does "Privacy Mode" do, and when would you disable it?
5. What `allowed_updates` list must you send to receive `chat_member` events?
6. How do you "kick" a user from a group via the bot API?
7. Why is the welcome flow gated with `restrictChatMember`?
8. What is the Telegram rate limit and how do you respect it?
9. What does `error_code: 403` mean in a `sendMessage` response?
10. How does the cron timezone option change the `0 8 * * *` spec?

Target: 8/10.

---

## Peer Coding Session

### Track A: /poll command

Add a `/poll` command that creates an inline-keyboard poll. Users click answers; the bot tracks votes. Show the current tally when a user asks `/results`.

### Track B: File uploads

Accept document uploads. Store them to disk or S3. When a user sends `/list` in a group, show the last 5 files shared with download links.

### Track C: Inline mode

A harder feature: inline bots respond when users type `@yourbot query` in any chat. Build one that lets any Telegram user search your product catalogue without opening a browser.

### Track D: Language detection

Detect the language of incoming messages (simple heuristic: Swahili words vs English) and reply in the same language using the `t()` helper from Week 13's USSD.

---

## Weekend Prep

The weekend project is a **group-management bot for a real community**. Pick a community you are part of (chama, cohort group, church WhatsApp group, etc) and build a Telegram bot tailored to their needs. Three typical use cases:

1. **Event RSVPs** -- `/rsvp` creates an event; members click Yes/No.
2. **Shared to-do list** -- `/todo add ...`, `/todo list`, `/todo done`.
3. **Finance tracker** -- contributions logged, balance queries, monthly summary.

You do not need all three. Pick one that solves a real problem for a real group.

Next week is `node-cron` in depth: handling failures, preventing overlap, deploying crons that survive restarts. Then Week 22 is Project 4 -- everything from Weeks 20-21 combined into the Chama Savings Platform.
