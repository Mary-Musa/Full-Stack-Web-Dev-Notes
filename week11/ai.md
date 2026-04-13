# Week 11 - AI Boundaries

**Ratio this week: 45% Manual / 55% AI**
**Habit introduced: "New tech = manual. Repeated patterns = AI."**
**Shift from last week: AI crosses past the halfway mark. This is the first week where you can honestly accept more AI help than manual work -- but only on the non-novel parts.**

This week introduces a new technology (WhatsApp Cloud API + Meta webhooks + conversation state machines) on top of patterns you already know (Express CRUD, React dashboards, database access). The habit is the split:

- **New tech (WhatsApp, webhooks, state machines):** manual first, AI after you have the shape.
- **Repeated patterns (CRUD routes, dashboard UI, search/filter/pagination):** AI-assisted from the start.

This is the rule real working engineers use. You are living it for the first time.

---

## Why The Ratio Crossed The Middle

By Week 11 you have built enough Express/React CRUD apps that doing another one from scratch is tedious and not instructive. You have already learned the shape. Doing it again by hand teaches you nothing new.

But WhatsApp Cloud API is new. Meta webhooks are new. Signature verification (HMAC-SHA256) is new. State machines for conversational bots are new. These parts need the "manual first" discipline, because your brain has no existing pattern for them. Skip the manual step on these and you will hit Week 12+ without the mental model for how chat bots and webhooks work.

The ratio is now a blend. Some of your code this week will be 80% manual (the webhook handler, the state machine). Some of it will be 20% manual (the dashboard routes, the leads table, the search UI). Average them out and you hit 45% -- roughly.

---

## What You Will Feel This Week

- The first WhatsApp message that echoes back from your bot will make you laugh out loud. Real phone. Real message. Real response. Actual magic.
- The webhook handshake (GET verify request) will fail the first three times and you will wonder if you typed your verify token wrong. You did.
- The state machine will feel elegant for a day, then tangled, then elegant again when you realise you are just handling conversation "turns".
- The CRM dashboard will come together fast because you have built similar dashboards before.
- You will have your first real "senior engineer moment": realising which parts of the codebase are worth your full attention and which parts are fine to scaffold.

---

## What You MUST Do Manually

### Day 1 -- WhatsApp Cloud API foundations (ALL MANUAL -- new tech)
- Read Meta's WhatsApp Cloud API documentation. Not a summary. Three pages minimum: Webhooks overview, Send Messages API, Receiving Messages.
- Set up your Meta Business account yourself. Register a test phone number. Get your permanent access token.
- Write the webhook verification endpoint (GET /webhook) by hand. Understand the `hub.mode`, `hub.verify_token`, `hub.challenge` query params.
- Test the handshake from the Meta dashboard. Fix it until it says "Verified".

### Day 2 -- Building the bot (ALL MANUAL -- new tech)
- HMAC signature verification on the webhook POST endpoint. Write this yourself using Node's `crypto` module. No AI.
- Parse Meta's inbound payload by hand. Know where `from`, `text.body`, `timestamp`, and `type` live in the JSON.
- Implement a state machine for the lead-capture flow: greeting -> name -> email -> inquiry type -> closing. Store session state per WhatsApp user in memory first, then in the database.
- Send replies back to WhatsApp using their Send API. Write the POST request yourself.

### Day 3 -- The CRM REST API (AI-ASSISTED -- repeated pattern)
- You have built CRUD APIs before (Week 6, Week 10). AI can scaffold this.
- **Still manual:** the endpoint that maps a WhatsApp number to a lead record. That mapping is the bridge between the bot and the CRM; it belongs to you.
- AI-scaffolded: the GET /leads list with pagination, filtering by status, search by name. Standard shapes you have seen.

### Day 4 -- The CRM dashboard (AI-ASSISTED -- repeated pattern)
- React dashboard that reads from your /leads endpoint. Dashboard layout, table, filters, status chips. Standard React patterns.
- AI can scaffold the UI components. You own the data-fetching hook (which you built in Week 9).
- Responsive layout, loading states, empty states, error states. All the React patterns you already know.

### Day 5 -- Week recap + weekend project
- Integrate the bot, the CRM API, and the dashboard into one working flow: WhatsApp in -> database write -> dashboard visible.
- End-to-end test: send a real message from your phone, watch it appear on the dashboard in real time.

---

## You Must Break Things On Purpose

- Temporarily remove the HMAC signature check. POST a fake webhook payload yourself and observe that your server accepts it. Restore the check.
- Put your bot into an infinite loop (send yourself a message; the bot replies; you reply; it replies forever). Add a guard.
- Send a malformed payload to your webhook. Observe your error handling.
- Break your state machine by jumping states out of order. Observe what your bot says.

Each one is a real production bug pattern.

---

## What You CAN Use AI For

### Full AI permissions on repeated-pattern work

For CRUD routes, dashboard tables, filter/search/pagination, styling, layout, error states, loading states -- AI is a full partner. Scaffold, review, refactor, extend. This is the same rhythm you would use in a real job.

### Restricted AI on new-tech work

For the WhatsApp webhook, the state machine, signature verification, payload parsing -- the discipline from Week 7's "new framework, manual first" habit applies. Write the first versions yourself. Once you have 2-3 state transitions working, AI can help you add new states.

### Good vs bad prompts this week

**Bad:** "Build me a WhatsApp lead-capture bot."
**Good:** "Here is my state machine with three states: greeting, name_prompt, email_prompt [paste]. I want to add a state for 'inquiry_type' after email. What is the smallest change to my machine to add it?"

**Bad:** "Write my CRM API."
**Good:** "I have my /leads endpoint working with pagination and status filter [paste]. I want to add full-text search across name and email. What is the cleanest SQL for this in PostgreSQL?"

**Bad:** "Make my dashboard pretty."
**Good:** "Here is my leads table [paste]. The status column is currently plain text. I want status chips with colours: new=blue, contacted=yellow, qualified=green, closed=grey. What is the minimal Tailwind class set for each?"

---

## The Split-Your-Attention Habit

Every feature this week, ask two questions:

1. **"Have I built this pattern before?"**
2. **"Is there a security or correctness cost if I get it wrong?"**

- Yes to 1, no to 2: AI-assisted is fine.
- Yes to 1, yes to 2: manual-first (payments, auth, webhooks).
- No to 1, no to 2: manual-first (learning).
- No to 1, yes to 2: manual-only (hard stop).

WhatsApp signature verification falls in "no to 1, yes to 2" this week. Dashboard pagination falls in "yes to 1, no to 2". The split is explicit; your AI Audit should show it.

---

## The 25-Minute Rule (With A Twist)

Same 25-minute rule as Week 8. But this week, the rule only applies to **new-tech code**. For repeated patterns you already know, the rule is shorter (10 minutes) because you should not be stuck for long on work you have already done three times.

If you find yourself stuck for 25 minutes on a CRUD pagination query you have written before, that is a flag: maybe you forgot the pattern. Revisit the earlier week's notes, not AI.

---

## Things AI Is Bad At This Week

- **Meta's API quirks.** Meta's docs are dense and the API has version numbers that mean nothing to AI. Always specify the version in your prompts: "using WhatsApp Cloud API v18".
- **Webhook signature math.** AI's crypto code sometimes uses the wrong hashing algorithm or missing encoding. Use Node's built-in `crypto.createHmac` and verify against Meta's documentation, not AI.
- **State machine structure.** AI will default to giant if/else chains. The state map pattern is cleaner. Know what shape you want before asking.
- **Your specific Meta account config.** AI does not know which test number you have, which callback URL you registered, or what your verify token is. Do not ask it to guess.

---

## Core Mental Models For This Week

- **A webhook is an inbound HTTP call you registered for.** The third party sends it when something happens. You verify it is real, then react.
- **HMAC signatures prove the sender knows a shared secret.** They do not encrypt anything. They are proof, not privacy.
- **A conversation is a finite state machine.** Each message from the user is an input. Each input moves the machine to the next state. The state is per-user.
- **A CRM is a database plus a dashboard plus channels.** Channels write to the database. The dashboard reads from it. Your job is to make those paths reliable.

---

## This Week's AI Audit Focus

Add a required section: **The split log.** For every feature you built, classify it:

```markdown
### Feature: webhook signature verification
- **Classification:** New tech, security cost. Manual-only.
- **Who wrote the code:** Me.
- **Did I use AI?** Only to explain HMAC after I wrote it.

### Feature: leads table with pagination
- **Classification:** Repeated pattern, no security cost. AI-assisted.
- **Who wrote the code:** AI scaffolded, I reviewed.
- **Changes I made to AI output:** Renamed a variable, tightened the error handler.
```

The facilitator reads the split. An audit where everything is "AI-assisted" (no manual-only entries) is a red flag -- it means the student skipped the new-tech learning.

---

## Assessment

Week 11 assessment is a whiteboard + code walkthrough:

- Draw the full data flow on a whiteboard: user -> WhatsApp -> Meta -> your webhook -> state machine -> database -> dashboard. No code. Just arrows.
- Walk through your signature verification code. Explain HMAC in thirty seconds.
- Walk through your state machine. What happens if a user sends "hello" while in state `awaiting_email`?
- Show your split log from the AI Audit. Defend the classification of any feature the facilitator points to.
- Live test: facilitator sends a real message to your bot from their phone. Everything should work end to end.

### Live Rebuild Check

Facilitator asks you to add a new state to your state machine. You do it on screen. The shape should match your existing pattern. If AI helps, the usage is logged.

---

## One Sentence To Remember

"Ask of every feature: is it new to me, and is it dangerous if wrong? The answer tells you how much AI to invite in." This is the split that defines senior engineering judgement.
