# Week 30 - AI Boundaries

**Ratio this week: 20% Manual / 80% AI**
**Habit introduced: "Engineer-led, AI-powered."**
**Shift from last week: The lowest manual ratio in the whole programme. This is the target state: you are a working engineer who happens to use AI extensively, not an AI user who happens to know some code.**

Week 30 is the final week. Testing, documentation, final deploy, demo rehearsal, and demo day. None of these are new concepts for you. All of them are things an engineer does in the last week before a launch. AI can legitimately do 80% of the typing. You own every decision, every verification, every demo moment.

The habit is the whole programme in one phrase: **engineer-led, AI-powered.** You drive. AI accelerates. That is the whole 30-week arc compressed into three words.

---

## Why 20/80

You have internalised every hard rule:

1. Never trust AI with money.
2. Never trust AI with auth.
3. Never trust AI with tenant isolation.
4. Never trust AI with architecture.
5. Design on paper first.
6. Read the real docs, not summaries.
7. Compare and contrast AI output with your own.
8. Audit every security-critical line.

With these rules active in your head, AI can safely do most of the typing. If you forget any of them under pressure, you will feel it -- and you know how to recover.

---

## What You MUST Do Manually (20%)

### Day 1 -- Testing and QA
- End-to-end tests for the critical paths: signup, checkout, payment, refund, multi-tenant isolation.
- Manual smoke test: walk the app as a new user and record every rough edge.

### Day 2 -- Documentation and runbook
- Write the README by hand. Explain what the SME OS is in three sentences, then setup instructions, then features.
- Write a RUNBOOK.md: how to deploy, how to rollback, how to read the logs, how to contact the owner.
- Both documents are your voice. AI can assist with code blocks in them, but the prose is yours.

### Day 3 -- Final deploy
- Deploy to production. Real domain, real TLS, real secrets.
- Watch the deploy logs. Fix anything that breaks.
- Smoke test production. Real money, small amounts.

### Day 4 -- Demo rehearsal
- Rehearse the demo three times. Get the timing right. Script the transitions.
- Prepare for the three hardest questions a judge might ask. Know your answers.

### Day 5 -- Demo day
- Demo the SME OS live. Walk through one full customer journey (USSD, WhatsApp, payment, admin).
- Handle questions. If you do not know an answer, say so. Do not bluff.

---

## What You CAN Use AI For (80%)

- **Test writing** (reviewed for coverage).
- **Documentation drafts** (rewritten for voice).
- **Deploy scripts** (reviewed line-by-line).
- **Demo rehearsal Q&A** (AI plays the sceptical judge; you practise answers).
- **Bug fixes** during demo week.

Forbidden:
- Writing the README in AI prose.
- Taking credit for AI code you did not understand.
- Letting AI answer a facilitator question through you.

---

## The Engineer-Led, AI-Powered Habit

You carry this habit into your first job, into every side project, into every interview. The shape:

1. **You decide what to build.**
2. **You design how to build it.**
3. **AI helps you build it faster.**
4. **You verify what got built.**
5. **You own the result.**

Missing step 1 or 2 makes you an AI user. Missing step 4 or 5 makes you a liability. All five together make you an engineer who ships.

---

## Things AI Is Bad At This Week

- **Confidence under pressure.** AI will sound confident whether it is right or wrong. Your job is to be slower and more accurate.
- **Demo presence.** AI cannot rehearse your demo for you. Live practice only.
- **Ownership.** If you cannot explain a line of code in the demo, you do not ship that line.

---

## Core Mental Models For The Final Week

- **Launching is a process, not a moment.** Testing, docs, deploy, rehearsal, demo -- each a step.
- **Your demo is your product.** A great product with a bad demo is invisible. A mediocre product with a great demo gets hired.
- **Ownership is the full-stack skill.** Every line of your repo is yours, whoever typed it.

---

## This Week's AI Audit Focus

The audit this week is a self-assessment: **Am I an engineer-led, AI-powered developer?**

Answer these questions honestly:
1. Can I explain every file in my repo?
2. Can I defend every architectural decision?
3. Can I rebuild any function from memory if I had to?
4. Can I spot a bug in AI-generated code?
5. Can I decide when AI is wrong?

Yes to all five = you graduated as intended. Yes to 3-4 = close, keep practising. Below 3 = revisit the habits from earlier weeks.

---

## Assessment: The Demo

The final assessment is the demo itself. You walk the facilitators (and any external guests) through your SME OS. You answer their questions. You ship.

Pass criteria:
- The demo works end to end on live infrastructure.
- You can explain every critical piece.
- You own every decision, even the AI-assisted ones.
- Your tenant isolation holds under attack.
- Money moves correctly.
- Auth holds.

---

## One Sentence To Remember

"Engineer-led, AI-powered. You drive; AI accelerates. Anything else is either ego or dependence."

---

## Graduation Note

You started at 90% manual, 10% AI. You end at 20% manual, 80% AI. The journey is not about reducing manual work -- it is about earning the right to reduce it by proving, week by week, that you understand what AI is doing on your behalf.

The graduates who thrive are not the ones who used the least AI or the most. They are the ones who used it with judgement. Every weekly habit you built is one axis of that judgement. Keep them. Use them. Teach them to the next person.

Good luck. Ship things. Help people.
