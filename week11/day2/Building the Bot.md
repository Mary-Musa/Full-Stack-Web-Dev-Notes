# Week 11, Day 2: Building the Bot

By the end of today, your WhatsApp number will hold a real conversation with anyone who messages it. The bot will greet them, ask for their name, ask for their email, send an interactive list for their inquiry type, confirm the details, and save a structured lead row to the database. The webhook will verify Meta's HMAC signature on every incoming request so nobody can inject fake leads. And the bot will handle the edge cases real users produce -- voice notes, emoji-only messages, typos, and the "let me start over" case.

This is the most conceptually dense day of the week. You are writing a state machine -- a tiny piece of software that tracks where a user is in a multi-turn conversation and decides what the next message means based on context. Every chat bot, every checkout flow, every onboarding wizard is one of these under the hood.

**Prior-week concepts you will use today:**
- `axios` for server-to-server HTTP (Week 10, Day 2)
- Prepared statements and `INSERT` / `UPDATE` / `SELECT` (Week 10, Day 1)
- Regular expressions for validation (Week 4)
- `switch` statements and control flow (Week 3)
- Try / catch and async / await (Week 6)
- `JSON.stringify` for audit logging (Week 5)

**Estimated time:** 4-5 hours

---

## Recap: What You Built Yesterday

On Day 1 you built:

- An Express server with CORS, JSON parsing, and a raw-body hook.
- A SQLite database with `leads`, `conversations`, and `messages` tables.
- A Meta Developer app with a WhatsApp test number, phone number ID, access token, and app secret.
- A verified webhook at `/webhook` that logs every incoming WhatsApp message.
- ngrok tunnelling your local server to a public HTTPS URL.

Today you replace the "log and return 200" POST handler with real bot logic and add signature verification on top. Before you start, confirm that sending "hi" from your phone to the test number still produces a JSON payload in your server terminal. If that is broken, fix it first -- nothing today will work without it.

---

## State Machines In One Minute

Your bot is always in exactly one **state**, and every incoming message either keeps it in the same state or **transitions** it to a new one. Each state knows two things:

1. What to send the user when we enter this state (the prompt).
2. What to do with the next message the user sends while we are in this state (the handler).

Our conversation has five states:

```
  awaiting_name   ->   awaiting_email   ->   awaiting_inquiry_type
                                                    |
                                                    v
                                              confirming
                                                    |
                                                    v
                                                complete
```

One "escape hatch" transition: if the user ever types `restart`, we jump back to `awaiting_name` and wipe their partial data.

### Why Not Just `if/else`?

You *could* write the bot as one giant `if (message === "hi") { ... } else if (message.includes("@")) { ... }` block. It works for thirty seconds. Then the user messages out of order. Then they send an emoji. Then they go quiet for an hour and come back. Then you add a sixth question and rewrite the whole thing.

A state machine makes order explicit. The *current state* decides what the next message *means*. A `"john@mail.com"` is a name if we are in `awaiting_name` (weird name, but fine) and an email if we are in `awaiting_email`. Context does the interpretation, not string matching.

### Why State Lives In The Database

Your Express server forgets everything between requests. When Meta's webhook fires five minutes later with the next message, your server has no memory of what happened earlier -- unless the state is persisted.

This is the central insight of state machine bots running on stateless HTTP servers: **the database holds the state; the code is just the transitions**. Every incoming webhook: load the state, decide what to do, write the new state back, send a reply. Do not try to keep anything in a global variable -- if the server restarts mid-conversation (nodemon, a crash, a deploy) you lose everything.

---

## The Outgoing Message Helper

Before we can write the bot, we need a helper to send messages *out* to Meta. This is the same shape as the `services/mpesa.js` file in Week 10 Day 2 -- all the `axios` calls to the third party in one file so the rest of our code stays clean.

Create `server/services/whatsapp.js`:

```javascript
// server/services/whatsapp.js
const axios = require("axios");

const GRAPH = "https://graph.facebook.com/v20.0";

function authHeaders() {
  return {
    Authorization: `Bearer ${process.env.META_ACCESS_TOKEN}`,
    "Content-Type": "application/json",
  };
}

async function sendText(to, body) {
  const url = `${GRAPH}/${process.env.META_PHONE_NUMBER_ID}/messages`;
  try {
    await axios.post(
      url,
      {
        messaging_product: "whatsapp",
        to,
        type: "text",
        text: { body },
      },
      { headers: authHeaders() }
    );
  } catch (err) {
    console.error("sendText failed:", err.response?.data || err.message);
    throw err;
  }
}

async function sendInquiryList(to) {
  const url = `${GRAPH}/${process.env.META_PHONE_NUMBER_ID}/messages`;
  try {
    await axios.post(
      url,
      {
        messaging_product: "whatsapp",
        to,
        type: "interactive",
        interactive: {
          type: "list",
          body: { text: "What are you interested in?" },
          action: {
            button: "Choose one",
            sections: [
              {
                title: "Inquiry type",
                rows: [
                  { id: "viewing", title: "Property viewing" },
                  { id: "test_drive", title: "Car test drive" },
                  { id: "quote", title: "Request a quote" },
                  { id: "billing", title: "M-Pesa / billing" },
                  { id: "other", title: "Something else" },
                ],
              },
            ],
          },
        },
      },
      { headers: authHeaders() }
    );
  } catch (err) {
    console.error("sendInquiryList failed:", err.response?.data || err.message);
    throw err;
  }
}

module.exports = { sendText, sendInquiryList };
```

Two functions: plain text, and an interactive list. Both wrap `axios.post` with proper error logging. The `err.response?.data` bit prints Meta's error JSON if the call fails -- that is where you will read messages like "recipient phone number not in allowed list".

### About Interactive Messages

Meta supports a few message types beyond plain text. You will use two this week:

- **Text** -- plain strings, up to 4096 characters. The default.
- **Interactive list** -- a message with a "tap to choose" button that opens a list of up to 10 options. Perfect for "What are you interested in?" -- much better than asking the user to type menu numbers.

There is a third category called **template messages** which you need when you want to *start* a conversation with a user who has not messaged you first. We do not use templates this week because our bot only *responds*. If you want to push campaigns later, templates are the thing to learn.

Meta also enforces a **24-hour customer service window**: once a user messages you, you can freely send text replies for the next 24 hours. After 24 hours of silence, you can only message them with an approved template. Our bot always replies within seconds, so this is not a problem -- but know it exists.

---

## The Bot Service

This file is the heart of the week. Create `server/services/bot.js`:

```javascript
// server/services/bot.js
const { v4: uuidv4 } = require("uuid");
const db = require("../db");
const { sendText, sendInquiryList } = require("./whatsapp");

// --- Persistence helpers --------------------------------------------------

function findOrCreateLead(waPhone, profileName) {
  let lead = db
    .prepare("SELECT * FROM leads WHERE wa_phone = ?")
    .get(waPhone);

  if (lead) return lead;

  const id = uuidv4();
  db.prepare(
    `INSERT INTO leads (id, wa_phone, name, status)
     VALUES (?, ?, ?, 'new')`
  ).run(id, waPhone, profileName || null);

  db.prepare(
    `INSERT INTO conversations (id, lead_id, state)
     VALUES (?, ?, 'awaiting_name')`
  ).run(uuidv4(), id);

  return db.prepare("SELECT * FROM leads WHERE id = ?").get(id);
}

function getConversation(leadId) {
  return db
    .prepare("SELECT * FROM conversations WHERE lead_id = ?")
    .get(leadId);
}

function setState(leadId, state) {
  db.prepare(
    `UPDATE conversations
     SET state = ?, last_message_at = datetime('now')
     WHERE lead_id = ?`
  ).run(state, leadId);
}

function updateLead(leadId, patch) {
  const fields = Object.keys(patch);
  if (fields.length === 0) return;
  const sets = fields.map((f) => `${f} = ?`).join(", ");
  const values = fields.map((f) => patch[f]);
  db.prepare(
    `UPDATE leads SET ${sets}, updated_at = datetime('now') WHERE id = ?`
  ).run(...values, leadId);
}

function logMessage(leadId, direction, body, rawPayload) {
  db.prepare(
    `INSERT INTO messages (lead_id, direction, body, raw_payload)
     VALUES (?, ?, ?, ?)`
  ).run(leadId, direction, body, JSON.stringify(rawPayload || null));
}

module.exports = {
  findOrCreateLead,
  getConversation,
  setState,
  updateLead,
  logMessage,
};
```

Five small helpers, each a thin wrapper around a single SQL statement. Nothing here is new -- you wrote functions exactly like these in Week 10 for links and payments. Keeping them in their own file means the state machine code below reads like *prose* instead of SQL soup.

A few design notes:

- **`findOrCreateLead` is the upsert pattern.** The same phone number should never create two lead rows, so we look first and insert only if missing. `wa_phone` has a `UNIQUE` constraint as a last line of defence.
- **`updateLead` builds its SQL dynamically from a patch object.** The same pattern you will use for the PATCH route tomorrow. It lets callers update only the fields they care about.
- **`logMessage` stringifies the whole raw payload.** If something goes wrong in production next month, that column is your time machine.

### Validation Helpers

Add these to the same file, above the `module.exports`:

```javascript
// server/services/bot.js  (add above module.exports)

// --- Validation -----------------------------------------------------------

function looksLikeEmail(str) {
  return typeof str === "string" && /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(str);
}

function looksLikeName(str) {
  return typeof str === "string" && str.trim().length >= 2;
}
```

These are intentionally loose. In a chat you do not want to be strict -- if someone types `aisha@gmail,com` (comma instead of dot), the right behaviour is to gently ask again, not to throw an error. Every invalid input is a chance to be patient, not strict. The bot is the business's first impression.

### The State Machine Itself

Still in `server/services/bot.js`, add the main handler. Update the `module.exports` at the bottom to export `handleIncoming`.

```javascript
// server/services/bot.js  (continuing)

// --- Conversation handler -------------------------------------------------

async function handleIncoming(message, contact) {
  const waPhone = message.from;
  const profileName = contact?.profile?.name;

  const lead = findOrCreateLead(waPhone, profileName);

  // Pull user-visible text out of whatever message type Meta sent
  let userText = null;
  let listChoiceId = null;

  if (message.type === "text") {
    userText = message.text?.body?.trim();
  } else if (message.type === "interactive") {
    const i = message.interactive;
    if (i?.type === "list_reply") {
      listChoiceId = i.list_reply.id;
      userText = i.list_reply.title;
    } else if (i?.type === "button_reply") {
      userText = i.button_reply.title;
    }
  } else {
    // voice note, image, sticker, location...
    await sendText(
      waPhone,
      "I can only read text messages right now. Could you type your answer?"
    );
    logMessage(lead.id, "inbound", `[${message.type}]`, message);
    return;
  }

  logMessage(lead.id, "inbound", userText, message);

  // Escape hatch: restart
  if (userText && /^(restart|start over|reset)$/i.test(userText)) {
    setState(lead.id, "awaiting_name");
    updateLead(lead.id, { name: null, email: null, inquiry_type: null });
    const reply = "No problem, let's start over. What's your full name?";
    await sendText(waPhone, reply);
    logMessage(lead.id, "outbound", reply);
    return;
  }

  const convo = getConversation(lead.id);

  switch (convo.state) {
    case "awaiting_name": {
      if (!userText || !looksLikeName(userText)) {
        const reply =
          "Hi! Welcome to Mctaba CRM. What's your full name? (at least 2 characters)";
        await sendText(waPhone, reply);
        logMessage(lead.id, "outbound", reply);
        return;
      }
      updateLead(lead.id, { name: userText });
      setState(lead.id, "awaiting_email");
      const reply = `Thanks ${userText.split(" ")[0]}! What's your email address?`;
      await sendText(waPhone, reply);
      logMessage(lead.id, "outbound", reply);
      return;
    }

    case "awaiting_email": {
      if (!looksLikeEmail(userText)) {
        const reply =
          "Hmm, that doesn't look like an email. Could you send it again? Example: yourname@gmail.com";
        await sendText(waPhone, reply);
        logMessage(lead.id, "outbound", reply);
        return;
      }
      updateLead(lead.id, { email: userText });
      setState(lead.id, "awaiting_inquiry_type");
      await sendInquiryList(waPhone);
      logMessage(lead.id, "outbound", "[interactive list: inquiry type]");
      return;
    }

    case "awaiting_inquiry_type": {
      if (!listChoiceId && !userText) {
        await sendInquiryList(waPhone);
        return;
      }
      const inquiry = listChoiceId || userText;
      updateLead(lead.id, { inquiry_type: inquiry });
      setState(lead.id, "confirming");

      const fresh = db
        .prepare("SELECT name, email, inquiry_type FROM leads WHERE id = ?")
        .get(lead.id);
      const reply =
        `Please confirm your details:\n\n` +
        `Name: ${fresh.name}\n` +
        `Email: ${fresh.email}\n` +
        `Inquiry: ${fresh.inquiry_type}\n\n` +
        `Reply "yes" to confirm or "restart" to start over.`;
      await sendText(waPhone, reply);
      logMessage(lead.id, "outbound", reply);
      return;
    }

    case "confirming": {
      if (/^y(es)?$/i.test(userText || "")) {
        setState(lead.id, "complete");
        const reply =
          "Thanks! Your details are saved. Someone from our team will be in touch shortly. To start a new inquiry, type 'restart'.";
        await sendText(waPhone, reply);
        logMessage(lead.id, "outbound", reply);
        return;
      }
      const reply =
        "Reply 'yes' to confirm the details above, or 'restart' to start over.";
      await sendText(waPhone, reply);
      logMessage(lead.id, "outbound", reply);
      return;
    }

    case "complete":
    default: {
      const reply =
        "Thanks! We already have your details. A team member will be in touch. To start a new inquiry, type 'restart'.";
      await sendText(waPhone, reply);
      logMessage(lead.id, "outbound", reply);
      return;
    }
  }
}

module.exports = {
  findOrCreateLead,
  getConversation,
  setState,
  updateLead,
  logMessage,
  handleIncoming,
};
```

Read this slowly. Every `case` in the `switch` is one state from the diagram at the top of the file; every `setState` call is a transition. The happy path flows top to bottom: `awaiting_name -> awaiting_email -> awaiting_inquiry_type -> confirming -> complete`. Every invalid input stays in the same state and re-asks.

### Edge Cases You Just Handled

| Case | What Meta Sends | What The Bot Does |
|---|---|---|
| User sends a voice note | `type: "audio"` | "I can only read text messages..." Stays in the same state. |
| User sends an image | `type: "image"` | Same -- asks for text. |
| User sends nothing but emoji | `type: "text"` but body is `""` after trim | Fails `looksLikeName` or `looksLikeEmail`, gets re-asked politely. |
| User types "restart" | text | State resets, data wiped, back to `awaiting_name`. |
| User messages again after `complete` | text | Polite "we already have your details" reply. |
| Two messages arrive within the same second | Two webhook calls back-to-back | Each call loads the current state from the DB, so they serialize naturally. SQLite's WAL mode (from Week 10) keeps this safe. |

You do not need to handle *every* possible case today. Get the happy path working first. Same approach that worked last week.

---

## Wiring The Webhook To The Bot

Now replace the placeholder POST handler in `server/routes/webhook.js`. This is the full file -- it replaces yesterday's version:

```javascript
// server/routes/webhook.js  (replaces yesterday's version)
const express = require("express");
const crypto = require("crypto");
const { handleIncoming } = require("../services/bot");

const router = express.Router();

// --- GET: verification handshake (unchanged from Day 1) -------------------
router.get("/", (req, res) => {
  const mode = req.query["hub.mode"];
  const token = req.query["hub.verify_token"];
  const challenge = req.query["hub.challenge"];

  if (mode === "subscribe" && token === process.env.META_VERIFY_TOKEN) {
    console.log("Webhook verified");
    return res.status(200).send(challenge);
  }
  return res.sendStatus(403);
});

// --- POST: real incoming events -------------------------------------------
router.post("/", async (req, res) => {
  if (!verifySignature(req)) {
    console.warn("Invalid webhook signature -- rejecting");
    return res.sendStatus(401);
  }

  // Acknowledge IMMEDIATELY. Meta retries if we take too long.
  res.sendStatus(200);

  try {
    const entry = req.body.entry?.[0];
    const change = entry?.changes?.[0];
    const value = change?.value;
    const messages = value?.messages;
    const contacts = value?.contacts;

    // Status updates (delivered/read) arrive without `messages`. Ignore.
    if (!messages || messages.length === 0) return;

    for (const message of messages) {
      const contact = contacts?.[0];
      await handleIncoming(message, contact);
    }
  } catch (err) {
    // We've already sent 200 -- log and move on.
    console.error("Error handling webhook:", err);
  }
});

// --- HMAC signature check -------------------------------------------------
function verifySignature(req) {
  const signature = req.headers["x-hub-signature-256"];
  if (!signature || !req.rawBody) return false;

  const expected =
    "sha256=" +
    crypto
      .createHmac("sha256", process.env.META_APP_SECRET)
      .update(req.rawBody)
      .digest("hex");

  try {
    return crypto.timingSafeEqual(
      Buffer.from(signature),
      Buffer.from(expected)
    );
  } catch {
    return false;
  }
}

module.exports = router;
```

Two big things are happening in this file.

**Immediate 200.** We acknowledge Meta *before* doing any work. If our bot logic throws or takes ten seconds, Meta still got its 200 and will not retry. The actual work happens in a `try/catch` after the response. This is standard webhook hygiene -- the same idea as the M-Pesa callback in Week 10, but this week we are more disciplined about it.

**Signature verification.** `crypto.createHmac` computes an HMAC-SHA256 of the raw request body using your app secret as the key. Meta did the same thing on their side and put the result in the `X-Hub-Signature-256` header. If they match, the request really came from Meta. This is the first time you have used cryptographic signatures in this course. You do not need to master HMAC theory today -- just know this is the correct production pattern, and that skipping it means anyone who guesses your webhook URL could inject fake leads.

`timingSafeEqual` is a standard library function that compares two buffers in a way that prevents a class of timing attacks. Use it instead of `===` whenever you are comparing secrets.

This is why we added the `verify` option on `express.json()` yesterday. Without it, `req.rawBody` would be undefined here and every signature check would fail.

---

## Testing The Full Conversation

Make sure the server is running (`npm run dev`) and ngrok is up. From your phone:

1. Send `hi`. You should get *"Hi! Welcome to Mctaba CRM. What's your full name?"*
2. Send `John Kamau`. You should get *"Thanks John! What's your email address?"*
3. Send `not-an-email`. You should get the polite re-ask.
4. Send `john@gmail.com`. You should get the interactive list popping up with five options.
5. Tap **Property viewing** (or any option). You should get a confirmation block showing your name, email, and inquiry.
6. Send `yes`. You should get the final thank-you.
7. Send another message. You should get *"Thanks! We already have your details..."*
8. Send `restart`. You should get *"No problem, let's start over..."* and the flow begins again.

✅ CHECKPOINT: If all eight steps work from a single phone conversation, Day 2 is complete.

Open `leads.db` with any SQLite viewer (the VS Code extension **SQLite Viewer** is a painless option) -- or you can just wait until tomorrow when the REST API will let you read it through HTTP. You should see:

- One row in `leads` with your name, email, `inquiry_type`, and `status = 'new'`.
- One row in `conversations` with `state = 'complete'` (or `awaiting_name` if you ended on a restart).
- Many rows in `messages` -- one for each inbound and outbound message, with `raw_payload` populated for inbound rows.

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `Invalid webhook signature -- rejecting` in the log | `META_APP_SECRET` in `.env` does not match the dashboard | Copy the App Secret from **App Settings -> Basic** in Meta -- click **Show** first -- and paste into `.env`. Restart the server. |
| The bot never replies but you see the webhook payload in the terminal | Outgoing API call is failing | Scroll up in the log for `sendText failed:` -- Meta's error JSON tells you exactly what is wrong. Usually an expired access token or a recipient not in the allowed list. |
| `{ "error": { "code": 190 } }` from Meta | Access token expired or malformed | Temporary tokens from the API Setup page last 24 hours. Generate a new one, paste into `.env`, restart. We make this permanent later in the week. |
| `{ "error": { "code": 131030 } }` | Recipient phone number not in the allowed list | On Meta's **API Setup** page, add the recipient's number under **To**. They must confirm via WhatsApp code. |
| Bot replies are going to the wrong number | `META_PHONE_NUMBER_ID` mismatch | Copy the Phone Number ID (not the display phone number) from the API Setup page into `.env`. These are different things. |
| `req.rawBody` is undefined | `express.json({ verify })` missing or in the wrong order in `index.js` | Make sure the `verify` option is set and `app.use(express.json(...))` appears *before* the route registrations. |
| Tapping a list option produces no response | `interactive.list_reply` branch missing | Double-check the block in `handleIncoming` that extracts `listChoiceId` from `message.interactive.list_reply`. |

Good time to commit: `git add . && git commit -m "feat: end-to-end bot conversation flow"`

---

## What You Built Today

- An outgoing message helper with text and interactive list support.
- A conversational state machine persisted in SQLite -- five states, clean transitions, a restart escape hatch.
- Validation that treats invalid input as a conversation, not an error.
- Handling for voice notes, images, and other non-text messages.
- HMAC-SHA256 signature verification on every incoming webhook using Node's built-in `crypto` module.
- Immediate 200 acknowledgement so Meta never retries while we are working.
- A full audit trail of every message in both directions in the `messages` table.

**Tomorrow:** You build the REST API that will power the dashboard. `GET /api/leads` with pagination, search, and status filtering. `GET /api/leads/:id` with the full conversation history. `PATCH /api/leads/:id` for sales team status updates. `GET /api/stats` for the summary cards. Everything you need to turn the database you have been filling up into a working sales tool. The React dashboard itself waits until Day 4.
