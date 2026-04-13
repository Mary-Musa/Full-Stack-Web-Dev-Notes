# Week 13 - AI Boundaries

**Ratio this week: 45% Manual / 55% AI**
**Habit introduced: "Protocol first, code second."**
**Shift from last week: Same ratio, but with a new-tech bump. USSD + Africa's Talking + Redis is three new technologies in one week.**

Last week closed Phase 2 reconciliation. This week opens Phase 2's final integration: USSD -- the only channel in the programme that reaches every Kenyan phone, including feature phones. You will build a working customer-support line a real business can put on a poster.

The new habit is specific to integration work: **understand the protocol before you write code for it.** USSD has quirks (session timeouts, the `text` field breadcrumb, `CON` vs `END`, the 182-character budget) that you will hit anyway. Learning them from Africa's Talking's actual documentation -- not from AI paraphrasing -- is the only way to avoid embarrassing bugs on demo day.

---

## Why The Ratio Dipped Back To 45/55

You hit 45/55 in Week 11 and stayed there in Week 12. Without the new-tech in Week 13 you might reasonably bump to 40/60. But three new technologies in one week is a lot: USSD sessions, Africa's Talking sandbox, and Redis session storage. Each one deserves manual-first treatment for its first hour.

By Friday the ratio effectively becomes 30/70 (scaffolding the dashboard and minor polish is pure AI work). The average across the week is 45/55.

---

## What You Will Feel This Week

- The first USSD prompt appearing on your actual phone will be the Week 13 equivalent of the Week 10 STK push moment: genuine magic.
- You will forget a `CON` prefix and wonder why your session ends after one screen. Five minutes of staring later, you add it and the session continues.
- The `text` breadcrumb ("") will confuse you for 30 minutes. Then you realise it is the full history separated by asterisks and the confusion evaporates.
- Redis will feel overkill for what you are doing. By Wednesday you will realise it is the only clean way to survive network blips mid-session.
- Writing the i18n helper on Day 4 will feel disproportionately satisfying.

---

## What You MUST Do Manually (45%)

### Day 1 -- USSD foundations (NEW TECH, MANUAL)
- Create an Africa's Talking sandbox account yourself. Read their USSD documentation (the real pages, not an AI summary).
- Write the first `/ussd` POST handler by hand. Understand the four fields AT sends: `sessionId`, `serviceCode`, `phoneNumber`, `text`.
- `CON` and `END` -- memorise the difference. Write one endpoint that always replies `END` and one that always replies `CON`. See the difference on your phone.
- Expose the endpoint via ngrok. Register the callback URL in the AT dashboard. Dial the service code from the simulator.

### Day 2 -- State machines and Redis sessions (NEW TECH, MANUAL)
- Install Redis yourself. Connect from Node. Write a one-line "ping" test.
- Design a state machine on paper: states, transitions, the shape of `context`. Pen before pixels.
- Write the dispatcher by hand. It reads the state, runs the handler, stores the new state. Maybe 40 lines total.
- Handlers are pure functions. Pass context in, return new context and response. No side effects. Write the first handler yourself.

### Day 3 -- Connecting USSD to the CRM
- This is the week's heavy lifting. Your USSD flow must write to the same Postgres database the Week 11-12 CRM uses.
- Manual work: the query that maps a phone number to a customer and the query that creates a new ticket. Because these touch the DB at the tenant boundary.
- AI-assisted work: the UI for the support dashboard showing USSD-originating tickets. You have built dashboards before.

### Day 4 -- Polishing for feature phones
- i18n helper: write `t(language, key)` yourself for at least 5 strings, in English and Kiswahili. Manual.
- Length clamp: every response capped at 182 characters. Write the clamp function yourself.
- Timeout handling: what happens when the user's session times out mid-flow? Write the reset path.
- Accessibility check: read every prompt out loud. If it is confusing to you, it will be confusing to a feature-phone user in Kawangware at 6am.

### Day 5 and weekend -- *384*CODE# Customer Support
- Build a working customer-support line for a real local business. End-to-end on a real phone.
- Must handle at least three flows: check status, new ticket, agent handoff.

---

## You Must Break Things On Purpose

- Forget to reply to an AT webhook within the timeout. Observe the "session terminated" response.
- Return 150 characters for a prompt that should be 80. Observe the phone's rendering quirks.
- Let the same `sessionId` hit your dispatcher twice in quick succession. Does your Redis state race?
- Break the i18n helper by removing a key. Fall back gracefully? Or crash? Either is fine -- know which.

---

## What You CAN Use AI For (55%)

Permissions:
- All previous permissions.
- **Dashboard scaffolding** for the CRM side (repeated pattern).
- **Refactoring the state machine** after you have the first version working.
- **Translating English strings to Kiswahili** (with a native-speaker review if possible -- AI Swahili is often too formal).

AI is forbidden for:
- Writing the first USSD handler.
- Writing the state dispatcher.
- Picking the shape of the state machine.
- The phone-number-to-customer mapping logic (tenant boundary).

### Good vs bad prompts this week

**Bad:** "Build me a USSD menu for a boda support line."
**Good:** "Here is my dispatcher [paste]. I have three states: welcome, track_ride, report_issue. I want to add a 'contact_support' state that presents a phone number. What is the minimal change to my state map?"

**Bad:** "Explain USSD to me."
**Good:** "I am reading AT's docs. The `text` field shows `1*2*3` after three navigation steps. Is that the full history of every key the user pressed, or just the selections?"

**Bad:** "Translate my USSD menu to Swahili."
**Good:** "Here are my English prompts [paste]. I want formal Kiswahili that a rural farmer in Bungoma would recognise -- not Sheng. Give me translations for each, and flag any where you are uncertain about the tone."

---

## The Protocol-First Habit

For every integration week from now on (USSD, Next.js, Stripe, Airtel, Telegram, BullMQ, Docker), the habit is:

1. Read the real documentation. Not AI-paraphrased. Not tutorials. The official docs.
2. Write down the protocol in your own words on paper. One page maximum.
3. Identify the quirks (things that break your usual assumptions).
4. Only then write code.

For USSD, the quirks are:
- Every response must start with `CON` or `END`.
- Sessions have hard timeouts (~180s).
- The `text` field is cumulative, not just the latest input.
- Character budget is ~182 per screen.
- Invalid menu options should re-render with `CON`, never `END`.

Writing these five quirks down *before* you code means you do not accidentally ship an app that ends the session on a typo.

---

## The 25-Minute Rule

Still. 25 minutes of real effort before asking AI on new-tech work. For repeated-pattern work (the dashboard side), 10 minutes is fine.

---

## Things AI Is Bad At This Week

- **AT's quirks.** Africa's Talking's docs have been updated many times. AI's training data is a mix of old and new. When AI says "AT returns X" verify with the real docs.
- **USSD length arithmetic.** AI frequently exceeds the 182-character budget. Always count the response length yourself in tests.
- **Redis key naming.** AI will give you verbose keys that hurt memory. Use short, scoped keys like `ussd:session:<id>`.
- **Kiswahili tone.** AI Swahili is usually too formal or mixes dialects. Get a human reviewer when possible.

---

## Core Mental Models For This Week

- **A USSD session is a short multi-step text conversation with hard deadlines.** Every screen is a hop; every hop has a budget.
- **`CON` continues, `END` ends.** Those two words are 90% of USSD.
- **State lives in Redis, not memory.** Node processes restart. Redis survives.
- **The simulator lies a little.** Test on a real phone before you trust any flow.

---

## This Week's AI Audit Focus

Add a required section: **Protocol quirks log.** List the quirks you discovered from the USSD docs *before* you coded:

```markdown
## USSD quirks (learned from AT docs, not AI)
1. Every response starts with CON or END.
2. Sessions expire after ~180s.
3. ...
```

If you cannot list five quirks, you did not read the docs deeply enough. Go back.

---

## Assessment

- Dial your code from a real phone. Walk the facilitator through a full flow end to end.
- Facilitator asks: "what happens if the user's session drops between step 2 and step 3?" You answer with the specific code path.
- Live task: add a new menu option that reads from Postgres (e.g., "my last 3 tickets"). You do it on screen.
- Show your protocol quirks log. Defend that you learned them from the docs, not AI.

### Live Rebuild Check

Facilitator deletes your state dispatcher. You rewrite it from memory. If you cannot, come back next week.

---

## One Sentence To Remember

"Every integration has quirks. The quirks are the whole job. Learn them from the source -- AI is a translator, not a surveyor."
