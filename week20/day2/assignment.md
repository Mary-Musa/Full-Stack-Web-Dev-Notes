# Week 20 - Day 2 Assignment

## Title
Inline Keyboards And Callback Queries

## Overview
Today you add inline keyboards -- the buttons that appear under bot messages -- and handle the callback queries they send when users tap them. This is how you build menu-driven flows in Telegram.

## Learning Objectives Assessed
- Send a message with an inline keyboard
- Receive callback queries
- Acknowledge the callback (or the button spins forever)
- Track button clicks per user

## Prerequisites
- Day 1 completed

## AI Usage Rules

**Ratio:** 40/60. **Habit:** New channel, manual handshake. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Scaffolding after your first keyboard works.
- **NOT ALLOWED FOR:** The first inline keyboard send.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Inline keyboard

**What to do:**
When the user sends `/menu`, reply with a keyboard:

```javascript
if (text === "/menu") {
  await axios.post(`${TELEGRAM_API}/sendMessage`, {
    chat_id: chatId,
    text: "Pick an option:",
    reply_markup: {
      inline_keyboard: [
        [{ text: "Products", callback_data: "action:products" }],
        [{ text: "My orders", callback_data: "action:orders" }],
        [{ text: "Support", callback_data: "action:support" }],
      ],
    },
  });
}
```

**Expected output:**
Three buttons visible under the message.

### Task 2: Handle callback queries

**What to do:**
Update your webhook to handle `callback_query` updates:

```javascript
const callbackQuery = req.body.callback_query;
if (callbackQuery) {
  const chatId = callbackQuery.message.chat.id;
  const data = callbackQuery.data;

  // Always acknowledge, even if you do not want to respond
  await axios.post(`${TELEGRAM_API}/answerCallbackQuery`, {
    callback_query_id: callbackQuery.id,
  });

  if (data === "action:products") {
    await axios.post(`${TELEGRAM_API}/sendMessage`, {
      chat_id: chatId,
      text: "Our products: ... (coming Day 3)",
    });
  }
  // etc
}
```

**Expected output:**
Tapping a button triggers a reply without making the button spinner hang.

### Task 3: Edit the original message

**What to do:**
Instead of sending a new message, edit the original one when a button is tapped:

```javascript
await axios.post(`${TELEGRAM_API}/editMessageText`, {
  chat_id: chatId,
  message_id: callbackQuery.message.message_id,
  text: "You picked Products. Loading...",
});
```

**Expected output:**
The menu message transforms instead of growing a new message chain.

### Task 4: Per-user state (Redis)

**What to do:**
Install ioredis (if not already). Track the user's current menu step in Redis with a TTL:

```javascript
await redis.set(`tg:state:${chatId}`, "products_list", "EX", 300);
```

Read it on the next message to know where they are.

**Expected output:**
State persists between messages.

### Task 5: Debug notes

**What to do:**
In `day2-notes.md`, write 4-5 sentences:
- What is the difference between a `message` update and a `callback_query` update?
- Why must you answer callback queries (`answerCallbackQuery`)?
- What happens if you do not?

**Expected output:**
Notes committed.

## Stretch Goals (Optional - Extra Credit)

- Add a ReplyKeyboard (persistent keyboard at the bottom) alongside inline.
- Add a "Back" button that returns to the main menu.
- Paginate a list of 20 products into multiple messages.

## Submission Requirements

- **What to submit:** Repo, screenshots, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 2

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Inline keyboard sent | 20 | Three buttons under a message. |
| Callback query handler | 25 | Responds to button taps. |
| answerCallbackQuery called | 10 | Spinner does not hang. |
| editMessageText used | 20 | Message transforms in place. |
| Redis state per user | 15 | TTL set correctly. |
| Clean commits | 10 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Forgetting to answer the callback query.** The Telegram client shows a loading spinner that never stops.
- **Trying to use the same message_id across chats.** Each chat has its own IDs.
- **Storing state in memory.** Node restarts wipe memory. Use Redis.

## Resources

- Day 2 reading: [Inline Keyboards and State.md](./Inline%20Keyboards%20and%20State.md)
- Week 20 AI boundaries: [../ai.md](../ai.md)
