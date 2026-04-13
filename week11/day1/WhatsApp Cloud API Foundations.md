# Week 11, Day 1: WhatsApp Cloud API Foundations

> **AI boundaries this week:** 45% manual / 55% AI. Habit: *New tech = manual. Repeated patterns = AI. Classify every feature before you code.* See [ai.md](../ai.md).

By the end of today, you will have an Express server running on your laptop that Meta's WhatsApp infrastructure can reach over a real HTTPS URL, a SQLite database with the three tables a CRM needs, and a verified webhook that logs every incoming WhatsApp message to your terminal. You will not have a working bot yet -- that is tomorrow. Today is about proving the wires are connected.

This is the start of a four-day capstone. By Friday you will have a WhatsApp bot that captures leads through a short conversation and a web dashboard where a sales team can search, filter, and manage them. The problem you are solving is real: a real estate agency in Lavington, a car yard on Ngong Road, or an insurance broker in Westlands can receive two hundred WhatsApp messages over a weekend and lose most of them because nobody can keep track of the chaos. Your system turns that chaos into a sales pipeline.

**Prior-week concepts you will use today:**
- Express, middleware, routes (Week 10, Day 1)
- SQLite with `better-sqlite3`, prepared statements, foreign keys (Week 10, Day 1)
- Environment variables and `.env` secrets (Week 10, Day 1)
- `.gitignore` and keeping credentials out of Git (Week 6)
- ngrok for exposing localhost over HTTPS (Week 10, Day 2 -- M-Pesa callbacks)
- The idea of a webhook: a URL on your server that a third party calls (Week 10, Day 2)

**Estimated time:** 3-4 hours

---

## The Week Ahead

Four days, one project. Here is how the work splits.

| Day | What You Build |
|---|---|
| Day 1 (today) | Project setup, Meta configuration, database schema, verified webhook that logs incoming messages. |
| Day 2 | The bot itself -- a state machine that holds a multi-turn conversation, collects name/email/inquiry, and replies over WhatsApp. HMAC signature verification. |
| Day 3 | The CRM REST API -- pagination, search, status filtering, lead detail with conversation history, stats endpoint. |
| Day 4 | The React + Tailwind dashboard -- stats cards, leads table, slide-out detail panel, one-click status updates, polling. Stretch goals. |

Each day builds directly on the previous one. If Day 1 is broken, Day 2 cannot work. Do not move on from today until the checkpoint at the end passes.

## Understanding The WhatsApp Business Platform

It is easy to get confused because Meta has three products with similar names.

| Product | What It Is | Use It For |
|---|---|---|
| **WhatsApp Business App** | A free Android/iOS app with a business profile and quick replies. Runs on one phone. | A shop owner replying by hand. |
| **WhatsApp Cloud API** | A programmable API hosted by Meta. HTTP requests in, webhooks out. | Any software product. What we use this week. |
| **Third-party providers (Twilio, Wati...)** | Companies that wrap the Cloud API in their own API and charge a markup. | Teams that do not want to deal with Meta directly. Not us. |

We go directly to Meta's Cloud API. It is free to start, there is no middleman, and the patterns you learn are the same patterns every paid provider wraps.

### How The Cloud API Works

Meta runs the servers that talk to every WhatsApp user in the world. You do not replace those servers -- you register your application with Meta, and Meta routes messages between your code and the users. The pieces you work with are:

- **App** -- A container in Meta's developer dashboard that holds your configuration.
- **Phone Number ID** -- Meta gives you a test phone number. Every API request you make says "I am sending this as that number." It is like a username for your bot.
- **Access Token** -- A long secret string you put in the `Authorization` header of every outgoing request. It proves you own the app. Treat it like an M-Pesa consumer secret -- never commit it.
- **Webhook** -- A URL on *your* server that Meta calls whenever a customer sends a message to your number. This is how your code hears from users.
- **Verify Token** -- A string *you* invent and type into both the Meta dashboard and your `.env` file. Meta uses it to prove a webhook registration request really came from you, the first time the webhook is wired up.

### The Request-Response Cycle, Reversed

In Week 10 your React app sent a `fetch()` to your Express server. Server listens, server responds. Easy.

A WhatsApp webhook reverses that. *Meta* sends a POST request to *your* server. Your server is now playing the role of the API, and Meta is playing the role of the client. Your job on that request is to (1) read the incoming JSON, (2) return a `200 OK` within a few seconds so Meta does not retry, and (3) make an outgoing `axios.post()` back to Meta's API to send the reply.

So every conversation turn is actually *two* HTTP cycles: Meta -- you, then you -- Meta. If you internalised the M-Pesa callback flow last week, this is the same shape -- Safaricom called your server when a payment completed; Meta calls your server when a user sends a message.

## System Architecture

Here is the full picture of what you are building this week. Pin it somewhere.

```
  +---------------------+
  |  Customer's phone   |   (regular WhatsApp, +254)
  +----------+----------+
             |
             | User types a message
             v
  +---------------------+
  |   Meta / WhatsApp   |
  |    Cloud API        |
  +----------+----------+
             |
             | HTTPS POST to your webhook URL
             v
  +---------------------+
  |  Your Express       |   (localhost:5000, exposed by ngrok)
  |  server (webhook)   |   - parses payload
  +----------+----------+   - loads conversation state
             |              - runs state machine (Day 2)
             |              - writes to SQLite
             | axios.post() back to Meta
             v
  +---------------------+
  |   Meta / WhatsApp   |
  +----------+----------+
             | Delivers reply
             v
  +---------------------+
  |  Customer's phone   |
  +---------------------+


  Separately, the sales team:

  +---------------------+    fetch /api/leads    +---------------------+
  |  React Dashboard    | ---------------------> |  Your Express       |
  |  (localhost:3000)   | <--------------------- |  server (REST API)  |
  +---------------------+     JSON response      +----------+----------+
                                                            |
                                                            v
                                                    +---------------+
                                                    |  SQLite       |
                                                    |  leads.db     |
                                                    +---------------+
```

The crucial insight: both sides use the **same Express server**. The webhook routes (`/webhook`) and the REST routes (`/api/leads`) live in one process, share one database connection, and deploy as one unit. This is the same server you built in Week 10 (where it held both `/api/links` and `/api/mpesa/callback` in one process). The shape is identical.

---

## Project Setup

### Folder Structure

Use the same two-folder layout from Week 10 -- one `server` folder, one `client` folder, each with their own `package.json`.

```bash
mkdir whatsapp-crm
cd whatsapp-crm
mkdir server
```

The React client waits until Day 4. Today is all about the server.

### Initializing The Backend

```bash
cd server
npm init -y
npm install express cors dotenv better-sqlite3 axios uuid
npm install --save-dev nodemon
```

Every one of these packages is something you installed in Week 10. We dropped `pdfkit` (no receipts this week) and kept everything else. This should look familiar from Week 10, Day 1 -- we are using the same setup.

Open `server/package.json` and add the dev script:

```json
{
  "scripts": {
    "dev": "nodemon index.js"
  }
}
```

### Environment Variables

Create `server/.env.example`. We are adding a small discipline this week -- commit an example file with empty values so anyone cloning your repo knows what variables they need. The real `.env` stays out of Git.

```env
# server/.env.example
PORT=5000

# Meta WhatsApp Cloud API
META_ACCESS_TOKEN=
META_PHONE_NUMBER_ID=
META_VERIFY_TOKEN=
META_APP_SECRET=

# App
APP_URL=http://localhost:3000
```

Copy it to the real file and fill in a verify token -- invent any random string you can remember. I will use `mctaba_crm_verify_2026` in examples. Yours should be different.

```env
# server/.env  (the rest gets filled in once you configure Meta)
PORT=5000
META_VERIFY_TOKEN=mctaba_crm_verify_2026
APP_URL=http://localhost:3000
```

Create `server/.gitignore`:

```
node_modules/
.env
leads.db
```

Same pattern as Week 10. Your M-Pesa keys would be in this `.env` on another project; your Meta access token goes in the same kind of file here.

Good time to commit: `git init && git add . && git commit -m "chore: scaffold server project"`

---

## The Database

Create `server/db.js`. Compare this side-by-side with the `db.js` from Week 10 -- the *shape* is identical; only the tables changed.

```javascript
// server/db.js
const Database = require("better-sqlite3");
const path = require("path");

const db = new Database(path.join(__dirname, "leads.db"));
db.pragma("journal_mode = WAL");

db.exec(`
  CREATE TABLE IF NOT EXISTS leads (
    id TEXT PRIMARY KEY,
    wa_phone TEXT NOT NULL UNIQUE,
    name TEXT,
    email TEXT,
    inquiry_type TEXT,
    status TEXT DEFAULT 'new',
    notes TEXT,
    assigned_to TEXT,
    created_at TEXT DEFAULT (datetime('now')),
    updated_at TEXT DEFAULT (datetime('now'))
  );

  CREATE TABLE IF NOT EXISTS conversations (
    id TEXT PRIMARY KEY,
    lead_id TEXT NOT NULL,
    state TEXT DEFAULT 'awaiting_name',
    last_message_at TEXT DEFAULT (datetime('now')),
    FOREIGN KEY (lead_id) REFERENCES leads(id)
  );

  CREATE TABLE IF NOT EXISTS messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    lead_id TEXT NOT NULL,
    direction TEXT NOT NULL,
    body TEXT,
    raw_payload TEXT,
    created_at TEXT DEFAULT (datetime('now')),
    FOREIGN KEY (lead_id) REFERENCES leads(id)
  );
`);

module.exports = db;
```

Three tables. You already know `CREATE TABLE` syntax from Week 10, so here is *why* the schema looks the way it does.

**`leads.wa_phone` is UNIQUE.** A WhatsApp phone number uniquely identifies a human. If the same person messages twice, we do not want two lead records. `UNIQUE` lets us look up existing leads by phone and avoid duplicates.

**`status` is a TEXT column, not a separate table.** The set of statuses is tiny and fixed -- `new`, `contacted`, `qualified`, `converted`, `lost`. A lookup table would be overkill. We enforce the allowed values in the Express route before writing.

**A separate `conversations` table.** A lead is a *person*; a conversation is a *session* with a state. Keeping them separate means we can reset the conversation (the user types "restart") without losing the lead.

**The `state` column lives in the database, not in memory.** Your Express server forgets everything between requests. When Meta's webhook fires five minutes later with the next message, your server has no memory of what happened earlier -- unless the state is persisted. This is the central insight of state machine bots: the database holds the state; the code is just the transitions. You will use this tomorrow.

**`messages` stores both `body` and `raw_payload`.** `body` is the clean text you show in the dashboard. `raw_payload` is the full JSON Meta sent, stringified. If Meta adds new fields next month or if you need to debug a weird failure, the raw payload is your audit trail. Same pattern as the `raw_callback` column in last week's `payments` table.

---

## The Express Server Entry Point

Create `server/index.js`:

```javascript
// server/index.js
require("dotenv").config();
const express = require("express");
const cors = require("cors");

const webhookRoutes = require("./routes/webhook");

const app = express();

// Middleware -- same as Week 10
app.use(cors());
// IMPORTANT: we need the raw request body later (Day 2) to verify Meta's
// HMAC signature. express.json's `verify` callback runs before parsing
// and lets us stash the raw bytes on req.rawBody.
app.use(
  express.json({
    verify: (req, _res, buf) => {
      req.rawBody = buf.toString("utf8");
    },
  })
);

app.use("/webhook", webhookRoutes);

// Placeholder routes for Days 3-4
app.use("/api/leads", (_req, res) =>
  res.status(501).json({ error: "Coming on Day 3" })
);
app.use("/api/stats", (_req, res) =>
  res.status(501).json({ error: "Coming on Day 3" })
);

app.get("/api/health", (_req, res) => {
  res.json({ status: "ok", timestamp: new Date().toISOString() });
});

app.use((err, _req, res, _next) => {
  console.error("Unhandled error:", err);
  res.status(500).json({ error: "Internal server error" });
});

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

The only line that is genuinely new compared to Week 10 is the `verify` option on `express.json()`. It runs a callback on every request *before* the body is parsed, giving us one chance to save the original bytes. By the time `express.json()` has parsed the JSON into `req.body`, the original bytes are gone. We need them tomorrow for signature verification, so we save them today.

Everything else -- `cors()`, JSON middleware, health check, error handler -- is straight out of Week 10.

---

## Registering With Meta

This is the fiddly part. Follow it step by step.

### Step 1: Create A Meta Developer Account And App

1. Go to `https://developers.facebook.com` and log in with your personal Facebook account. If you do not have one, create one -- Meta requires a real account for developer access.
2. Accept the developer terms when prompted.
3. Click **My Apps** in the top right, then **Create App**.
4. When asked for a use case, pick **Other**.
5. For app type, choose **Business**.
6. Give the app a name (e.g. `Mctaba CRM`), enter your email, and click **Create App**. You may be asked for your Facebook password.

### Step 2: Add WhatsApp To The App

1. On the app's dashboard, find **Add products to your app** and click **Set up** on the **WhatsApp** card.
2. When asked to link a Business Account, either pick an existing one or let Meta create one automatically.
3. You will land on a page called **API Setup**. This is your home base.

On that page Meta gives you:

- A **test phone number** (something like `+1 555 123 4567`). This is the number your bot sends *from*.
- A **Phone Number ID**. Copy it into `META_PHONE_NUMBER_ID` in your `.env`.
- A **temporary access token** (valid 24 hours). Copy it into `META_ACCESS_TOKEN`. We will swap it for a permanent token later this week.
- A box to **add recipient phone numbers**. Add your own Kenyan number here (`+254...`). Meta will send a code via WhatsApp; enter the code to verify. On the sandbox you can add up to five recipients.

Under **App Settings -> Basic**, find the **App Secret** field, click **Show**, and copy it into `META_APP_SECRET`. We use it tomorrow.

Your `.env` should now look like this (values redacted):

```env
# server/.env
PORT=5000

META_ACCESS_TOKEN=EAAG...very-long-string
META_PHONE_NUMBER_ID=123456789012345
META_VERIFY_TOKEN=mctaba_crm_verify_2026
META_APP_SECRET=a1b2c3d4e5f6...

APP_URL=http://localhost:3000
```

⚠️ COMMON ERROR: If you paste the access token and it has a trailing space, every outgoing call tomorrow will fail with `{ "error": { "code": 190 } }`. Paste carefully.

### Step 3: Start ngrok

You used this in Week 10 for M-Pesa callbacks. Same idea. In a *new* terminal (keep it running alongside your server):

```bash
ngrok http 5000
```

ngrok prints a line like:

```
Forwarding  https://a1b2-41-90-12-34.ngrok-free.app -> http://localhost:5000
```

Copy the `https://...ngrok-free.app` URL. Your webhook URL is that URL + `/webhook`.

One gotcha: on the free tier, the ngrok URL changes every time you restart ngrok. Each time it changes, you must update the webhook URL in the Meta dashboard. Annoying but free.

---

## The Webhook Verification Handshake

Before Meta will send real messages to your URL, it needs to confirm you actually control it. The handshake works like this: Meta sends a GET request to your webhook URL with three query parameters. If you echo one of them back as plain text, Meta accepts the webhook. One-time flow.

Create `server/routes/webhook.js`:

```javascript
// server/routes/webhook.js
const express = require("express");
const router = express.Router();

// GET /webhook -- Meta's verification handshake.
// Meta hits this once when you first register the webhook URL.
router.get("/", (req, res) => {
  const mode = req.query["hub.mode"];
  const token = req.query["hub.verify_token"];
  const challenge = req.query["hub.challenge"];

  if (mode === "subscribe" && token === process.env.META_VERIFY_TOKEN) {
    console.log("Webhook verified");
    return res.status(200).send(challenge);
  }

  console.warn("Webhook verification failed", { mode, token });
  return res.sendStatus(403);
});

// POST /webhook -- real incoming messages.
// For today we just log the payload. Tomorrow we wire it up to the bot.
router.post("/", (req, res) => {
  console.log("Incoming webhook:", JSON.stringify(req.body, null, 2));
  res.sendStatus(200);
});

module.exports = router;
```

The GET handler is a *verification challenge*. Meta wants to prove two things: (1) you control this URL, and (2) you know the verify token. It sends `hub.mode=subscribe`, `hub.verify_token=whatever-you-set`, and `hub.challenge=<random-string>`. You reply with the random string as plain text, and Meta accepts the webhook. You only see this flow once per registration.

The POST handler today just logs the incoming payload and returns 200. That is enough to prove the wiring is working. Tomorrow we replace it with the real bot logic.

---

## Registering The Webhook URL With Meta

Back in the Meta dashboard, on the **WhatsApp -> Configuration** page:

1. Start your server: `npm run dev`. Make sure ngrok is still running in its terminal.
2. In the **Webhook** section click **Edit**.
3. **Callback URL**: `https://<your-ngrok-url>.ngrok-free.app/webhook`
4. **Verify token**: whatever you put in `META_VERIFY_TOKEN`. They must match *exactly*.
5. Click **Verify and save**.

If it works, your server logs `Webhook verified` and Meta closes the modal.

6. Still on the Configuration page, find the **Webhook fields** section and click **Manage**. Subscribe to the `messages` field. Without this, Meta will verify your URL but never send real messages to it.

⚠️ COMMON ERROR: If Meta says *"The callback URL or verify token couldn't be validated"*, check in this order:
1. Your server is running (`npm run dev` in its terminal).
2. ngrok is running and the URL in Meta matches the *current* ngrok URL exactly, including `/webhook` at the end.
3. Your `META_VERIFY_TOKEN` in `.env` matches what you typed in the dashboard character-for-character.
4. You restarted the server after editing `.env` -- nodemon does this automatically, but confirm.

---

## Testing End-To-End

Start your server if it is not already running. Make sure ngrok is up. On Meta's API Setup page there is a button that opens WhatsApp on your phone with a chat pre-filled to the test number -- or you can save the test number manually and message it.

From your phone, send `hello`.

Look at your server terminal. You should see a large JSON payload logged that looks roughly like this:

```json
{
  "object": "whatsapp_business_account",
  "entry": [
    {
      "id": "WABA_ID",
      "changes": [
        {
          "field": "messages",
          "value": {
            "messaging_product": "whatsapp",
            "metadata": {
              "display_phone_number": "15551234567",
              "phone_number_id": "PHONE_NUMBER_ID"
            },
            "contacts": [
              { "profile": { "name": "Aisha" }, "wa_id": "254712345678" }
            ],
            "messages": [
              {
                "from": "254712345678",
                "id": "wamid.ABC...",
                "timestamp": "1712760000",
                "type": "text",
                "text": { "body": "hello" }
              }
            ]
          }
        }
      ]
    }
  ]
}
```

Study this payload. Tomorrow you will read specific fields out of it:

- `entry[0].changes[0].value.messages[0].from` -- the customer's WhatsApp phone (no `+`, no `0`).
- `entry[0].changes[0].value.messages[0].type` -- `text`, `image`, `audio`, `interactive`, etc.
- `entry[0].changes[0].value.messages[0].text.body` -- the text they sent.
- `entry[0].changes[0].value.contacts[0].profile.name` -- the WhatsApp display name of the sender.

**Checkpoint -- do not move to Day 2 until all of these pass:**

1. `npm run dev` starts the server without errors.
2. `ngrok http 5000` is running in a separate terminal.
3. Meta's dashboard shows the webhook as verified and subscribed to `messages`.
4. Sending "hello" from your phone to the test number causes a JSON payload to appear in your server terminal within a few seconds.
5. Your `.env` contains `META_ACCESS_TOKEN`, `META_PHONE_NUMBER_ID`, `META_VERIFY_TOKEN`, and `META_APP_SECRET`.

Good time to commit: `git add . && git commit -m "feat: webhook verification working"`

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `Cannot find module './routes/webhook'` | File path wrong | Check `server/routes/webhook.js` exists and the path in `require()` matches |
| Meta says webhook verification failed | Verify token mismatch | Copy `META_VERIFY_TOKEN` from `.env` into the Meta dashboard character by character |
| Meta verifies the URL but no messages arrive when you send from your phone | You did not subscribe to the `messages` field | In **Configuration -> Webhook fields**, click **Manage** and subscribe to `messages` |
| ngrok says "tunnel session failed" or demands a signup | Free tier quota or not signed in | Create a free ngrok account, add your authtoken with `ngrok config add-authtoken <token>` |
| Server starts but Meta cannot reach the URL | ngrok restarted since you last updated the dashboard | ngrok URLs change on restart on the free tier. Copy the new URL into Meta's webhook config. |
| `Error: listen EADDRINUSE :::5000` | Port 5000 already in use | Stop the other process, or change `PORT` in `.env` to `5001` and restart ngrok as `ngrok http 5001` |

---

## What You Built Today

- A Node.js project with the same folder structure and `.env` pattern as Week 10.
- A SQLite database with three tables ready for a CRM: `leads`, `conversations`, `messages`.
- An Express server with CORS, JSON middleware, and a raw-body hook for tomorrow's signature verification.
- A Meta Developer app with WhatsApp Cloud API configured and your phone registered as a test recipient.
- A public HTTPS tunnel via ngrok pointing at your laptop.
- A verified webhook that logs every incoming WhatsApp message to your terminal.

**Tomorrow:** You turn that webhook into a real bot. Incoming payloads will be parsed, the state machine will run, outgoing replies will be sent via Meta's API, and you will implement HMAC signature verification so no one can inject fake leads into your server. By the end of Day 2, a friend messaging your test number will be able to complete a full name-email-inquiry conversation and you will see a finished lead row in the database.
