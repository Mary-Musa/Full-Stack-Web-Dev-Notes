# Week 11 - Day 1 Assignment

## Title
Wire Up The WhatsApp Cloud API Webhook

## Overview
Week 11 builds a WhatsApp Lead Capture CRM. Today you set up Meta Developer account access, register a test number, expose your Express server via ngrok, and complete Meta's webhook verification handshake. By end of day, Meta's servers can POST inbound WhatsApp messages to your local machine and your server responds correctly.

## Learning Objectives Assessed
- Register for Meta Developer and create a WhatsApp Cloud API app
- Understand Meta's webhook verification (GET handshake with hub.challenge)
- Write an Express route that answers the verification and accepts POST updates
- Log inbound WhatsApp payloads to understand their shape

## Prerequisites
- Week 10 completed
- Your `payments-backend` or a new Express project running locally

## AI Usage Rules

**Ratio this week:** 45% manual / 55% AI
**Habit:** New tech = manual. Repeated patterns = AI. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Dashboard UI scaffolding later in the week (repeated pattern).
- **NOT ALLOWED FOR:** Writing your first webhook verification route. Writing your HMAC signature check. Parsing Meta's payload by hand.
- **AUDIT REQUIRED:** Yes. Include a "Split log" classifying every feature as new-tech-manual or repeated-pattern-AI.

## Tasks

### Task 1: Create a Meta Developer app

**What to do:**
1. Go to https://developers.facebook.com/ and sign up or log in.
2. Create a new app, type "Business".
3. Add the "WhatsApp" product to the app.
4. Meta gives you a test phone number + a temporary access token. Copy both into your `.env` file as `META_PHONE_NUMBER_ID` and `META_ACCESS_TOKEN`.
5. Pick a verify token of your own (any string) and add it to `.env` as `META_VERIFY_TOKEN`.

**Expected output:**
All three env variables set. Screenshot `day1-meta-app.png` of your app dashboard.

### Task 2: Write the verification handshake by hand

**What to do:**
In a new project folder `whatsapp-crm/` (or your existing Express backend), create `routes/whatsapp.js`:

```javascript
const express = require("express");
const router = express.Router();

router.get("/webhook", (req, res) => {
  const mode = req.query["hub.mode"];
  const token = req.query["hub.verify_token"];
  const challenge = req.query["hub.challenge"];

  if (mode === "subscribe" && token === process.env.META_VERIFY_TOKEN) {
    console.log("Webhook verified");
    return res.status(200).send(challenge);
  }
  return res.sendStatus(403);
});

module.exports = router;
```

Mount it in `index.js`:

```javascript
const whatsappRoutes = require("./routes/whatsapp");
app.use("/whatsapp", whatsappRoutes);
```

Every line hand-typed. No AI.

**Expected output:**
GET /whatsapp/webhook with the right query params returns the challenge.

### Task 3: Register the webhook with Meta

**What to do:**
1. Run your server and expose port 3001 via ngrok: `ngrok http 3001`
2. Copy the public HTTPS URL.
3. In Meta dashboard > WhatsApp > Configuration, paste `https://YOUR_NGROK/whatsapp/webhook` as the Callback URL and enter your verify token.
4. Click "Verify and save". Meta hits your GET endpoint. You see "Webhook verified" in your server logs.
5. Subscribe to the `messages` field on the WABA.

**Expected output:**
Meta shows the webhook as verified. Screenshot `day1-verified.png`.

### Task 4: Accept POST updates and log them

**What to do:**
Add a POST handler alongside the GET:

```javascript
router.post("/webhook", express.json(), (req, res) => {
  console.log("Incoming WhatsApp update:", JSON.stringify(req.body, null, 2));
  res.sendStatus(200);
});
```

From your phone, send a WhatsApp message to the test number Meta provided. Your terminal should log the full payload.

**Expected output:**
A real message appears in your server logs. Screenshot `day1-inbound.png`.

### Task 5: Write the split log

**What to do:**
In `AI_AUDIT.md`, start a new "Split log" section. For the two features you built today (webhook verification, payload logging), classify each:

```markdown
### Feature: Webhook verification
- Classification: NEW TECH -- manual-first.
- Who wrote the code: Me. Hand-typed, no AI.

### Feature: Payload logging middleware
- Classification: Repeated pattern (Express middleware).
- Who wrote the code: Me, but AI would be acceptable. I chose manual because the file is small.
```

**Expected output:**
Audit updated.

## Stretch Goals (Optional - Extra Credit)

- Persist every inbound payload to a `logs/inbound.jsonl` file (one JSON per line).
- Add a simple filter that ignores message types other than `text`.
- Use the GET handshake pattern in a separate test endpoint that just echoes a challenge.

## Submission Requirements

- **What to submit:** Repo with `routes/whatsapp.js`, screenshots, `AI_AUDIT.md`.
- **Deadline:** End of Day 1

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Meta app created and subscribed to messages | 15 | Test number working, verify token in .env. |
| Verification route by hand | 25 | GET handler works. No AI in the file. |
| Meta shows webhook as verified | 20 | Screenshot confirms. |
| POST handler logs inbound messages | 20 | Real WhatsApp message appears in logs. |
| Split log in audit | 15 | Both features classified with reasoning. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Returning 404 on the GET challenge.** Meta sends GET before POST. Your route must handle both methods on the same path.
- **Skipping the verify token check.** Without it, anyone could register your webhook. Always compare against `META_VERIFY_TOKEN`.
- **Letting AI write the handshake.** This is new-tech; the audit will flag it.

## Resources

- Day 1 reading: [WhatsApp Cloud API Foundations.md](./WhatsApp%20Cloud%20API%20Foundations.md)
- Week 11 AI boundaries: [../ai.md](../ai.md)
- Meta WhatsApp webhooks: https://developers.facebook.com/docs/whatsapp/cloud-api/webhooks
