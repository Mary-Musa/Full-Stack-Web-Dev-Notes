# Week 11 - Day 2 Assignment

## Title
Build The Lead-Capture Bot State Machine

## Overview
Today you build the conversational bot that greets new WhatsApp senders, asks for their name, email, and inquiry, and stores each conversation as a lead in your database. You also add HMAC signature verification to your webhook so Meta is the only sender you trust.

## Learning Objectives Assessed
- Verify Meta webhook HMAC signatures using Node's crypto module
- Parse Meta's inbound message payload
- Implement a per-user state machine for a multi-turn conversation
- Store conversation state in memory first, then in SQLite

## Prerequisites
- Day 1 completed

## AI Usage Rules

**Ratio:** 45/55. **Habit:** New tech = manual. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Helping extend a working state handler with a new step AFTER your base machine has 3+ states working.
- **NOT ALLOWED FOR:** Writing the signature verification. Writing the first three state transitions. Designing the state shape.
- **AUDIT REQUIRED:** Yes. Split log extended.

## Tasks

### Task 1: HMAC signature verification

**What to do:**
Meta signs every POST with `x-hub-signature-256`. Verify it by hand using Node's `crypto`:

```javascript
const crypto = require("crypto");

function verifySignature(req, res, buf) {
  const signature = req.headers["x-hub-signature-256"];
  if (!signature) {
    throw new Error("No signature");
  }
  const expected = "sha256=" + crypto
    .createHmac("sha256", process.env.META_APP_SECRET)
    .update(buf)
    .digest("hex");
  if (!crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
    throw new Error("Invalid signature");
  }
}

app.use("/whatsapp/webhook", express.json({ verify: verifySignature }));
```

Grab your app secret from Meta dashboard > App Settings > Basic. Set `META_APP_SECRET` in `.env`.

Type every line yourself. No AI.

**Expected output:**
Valid Meta POSTs pass. Tampered POSTs (edit one character of the body via curl) are rejected.

### Task 2: Parse the inbound payload

**What to do:**
Meta's payload is deeply nested. Extract what you need:

```javascript
function parseIncoming(body) {
  const entry = body.entry?.[0];
  const change = entry?.changes?.[0];
  const value = change?.value;
  const message = value?.messages?.[0];
  if (!message) return null;

  return {
    from: message.from,            // phone number
    text: message.text?.body,      // text content
    timestamp: message.timestamp,
    messageId: message.id,
  };
}
```

Test with a real message. Log the parsed object.

**Expected output:**
Parsed object with `from`, `text`, `timestamp` visible.

### Task 3: State machine with three states

**What to do:**
Define a simple in-memory state machine that tracks per-phone-number progress:

```javascript
const STATES = {
  GREETING: "greeting",
  AWAITING_NAME: "awaiting_name",
  AWAITING_EMAIL: "awaiting_email",
  AWAITING_INQUIRY: "awaiting_inquiry",
  DONE: "done",
};

const sessions = new Map(); // phone -> { state, data }

function handleMessage({ from, text }) {
  let session = sessions.get(from) || { state: STATES.GREETING, data: {} };

  if (session.state === STATES.GREETING) {
    session.state = STATES.AWAITING_NAME;
    sessions.set(from, session);
    return "Karibu! What is your full name?";
  }
  if (session.state === STATES.AWAITING_NAME) {
    session.data.name = text;
    session.state = STATES.AWAITING_EMAIL;
    sessions.set(from, session);
    return `Asante ${text}. What is your email?`;
  }
  if (session.state === STATES.AWAITING_EMAIL) {
    session.data.email = text;
    session.state = STATES.AWAITING_INQUIRY;
    sessions.set(from, session);
    return "What are you interested in?";
  }
  if (session.state === STATES.AWAITING_INQUIRY) {
    session.data.inquiry = text;
    session.state = STATES.DONE;
    sessions.set(from, session);
    // TODO: persist to DB tomorrow
    return "Thank you. An agent will be in touch shortly.";
  }
  return "You are all set. Dial again any time.";
}
```

Every line hand-typed.

**Expected output:**
Four turns of conversation work correctly for a single phone number.

### Task 4: Send replies back via Meta API

**What to do:**
Write the send function:

```javascript
const axios = require("axios");

async function sendMessage(to, text) {
  await axios.post(
    `https://graph.facebook.com/v18.0/${process.env.META_PHONE_NUMBER_ID}/messages`,
    {
      messaging_product: "whatsapp",
      to,
      text: { body: text },
    },
    {
      headers: { Authorization: `Bearer ${process.env.META_ACCESS_TOKEN}` },
    }
  );
}
```

Wire your POST handler to call `handleMessage` then `sendMessage`.

**Expected output:**
Messaging your bot produces real replies on your phone.

### Task 5: Run a full happy-path conversation

**What to do:**
From your phone, start a fresh conversation with the bot. Send four messages to complete the flow. Screenshot the WhatsApp conversation as `day2-conversation.png`.

**Expected output:**
Screenshot showing greeting -> name -> email -> inquiry -> thank you.

## Stretch Goals (Optional - Extra Credit)

- Validate the email with a regex. Re-prompt if invalid.
- Add a "restart" command that resets the state if the user types "restart".
- Add a timeout: if no reply for 10 minutes, reset to GREETING.

## Submission Requirements

- **What to submit:** Repo, screenshots, updated `AI_AUDIT.md`.
- **Deadline:** End of Day 2

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| HMAC signature verification | 25 | Valid requests pass, tampered ones rejected. No AI in this file. |
| Payload parser | 15 | Cleanly extracts from/text/timestamp. |
| State machine with 4+ states | 25 | All transitions work for a single user. |
| Real WhatsApp replies | 15 | Bot actually replies. Screenshot confirms. |
| Split log updated | 15 | Every feature classified correctly. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Using `===` for signature comparison.** Use `crypto.timingSafeEqual`. A naive comparison leaks timing info.
- **Sharing state across users.** The `sessions` Map must be keyed by phone number. One user should never see another user's progress.
- **Forgetting to read the body before verifying.** The `express.json({ verify: ... })` option lets you read the raw body for the signature check.

## Resources

- Day 2 reading: [Building the Bot.md](./Building%20the%20Bot.md)
- Week 11 AI boundaries: [../ai.md](../ai.md)
- Meta webhook security: https://developers.facebook.com/docs/graph-api/webhooks/getting-started#verification
