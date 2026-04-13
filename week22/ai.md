# Week 22 - AI Boundaries

**Ratio this week: 25% Manual / 75% AI**
**Habit introduced: "Model the domain yourself."**
**Shift from last week: The highest AI ratio so far. Project 4 (Chama Savings Platform) is almost entirely composition of known patterns.**

Project 4 is a big project but a familiar one. Telegram bot (Week 20), cron (Week 21), payments (Weeks 10-18), database schema (Week 12), React dashboard (many weeks). You have done every component of this project before, just not together.

The habit is narrow: the *domain* -- what a chama is, what states it moves through, how contributions accumulate, what fines look like -- is yours to model. AI can scaffold the code once the model is clear.

---

## What You MUST Do Manually (25%)

### Day 1 -- Project 4 architecture
- Draw the full system on paper. Chama as an aggregate: members, contribution schedule, payouts, fines, cycles.
- Decide the state machine for a chama's lifecycle (active -> rollover -> payout -> next cycle).
- Decide the state machine for a contribution (pending -> paid -> late -> overdue -> fined).
- This is a 2-3 hour design session. No code. No AI.

### Day 2 -- Chama bot core
- First handshake on a real test group (repeat Week 20 habit).
- Create the core chama commands: `/start`, `/join`, `/pay`, `/balance`, `/rules`. These you write by hand because they establish the pattern.

### Day 3 -- Contributions via M-Pesa
- The payment flow reuses the Week 10 M-Pesa code. The money rule still applies.
- Manual: the amount computation for a contribution (scheduled amount + any fines). Manual: the idempotency key.
- AI-assisted: the UI for confirming payment in the bot.

### Day 4 -- Cycles, fines, reminders
- Cron jobs from Week 21: close a cycle, compute fines for overdue members, send reminders. Reuse the cron patterns.
- Manual: the fine computation rule (must match the chama's written rules exactly).

### Day 5 -- Treasurer dashboard
- A small Next.js admin dashboard for the treasurer. Almost entirely reused patterns from Weeks 14-16.

---

## What You CAN Use AI For (75%)

- All integrations (reuse from prior weeks).
- All UI (dashboard, bot menus).
- All boilerplate.
- Tests.

Forbidden:
- Modelling the domain.
- Fine computation rules (business rule).
- Payout state machine (money rule).

---

## Things AI Is Bad At This Week

- **The specific rules of your chama.** Real chamas have quirks: "members who join mid-cycle contribute pro-rata", "the chairperson contributes double", etc. AI will default to textbook rules. Use your actual group's rules.
- **Edge cases in cycles.** What if a member leaves mid-cycle? What if a payment arrives late by 2 days -- fine or grace period? Decide yourself.

---

## Core Mental Models For This Week

- **A chama is a temporal aggregate.** It has a schedule. Everything ties back to the schedule.
- **A cycle is a unit of the schedule.** At the end of a cycle, the state transitions.
- **Fines are a derived value** computed from lateness, not stored. (Unless you need historical accuracy, in which case store them at the moment they apply.)

---

## This Week's AI Audit Focus

Include your paper-drawn domain model (photo or ASCII) at the top of the audit. This is the single most important artefact of the week.

---

## Assessment

- Live demo: set up a test chama with three members. Walk through a full cycle: contributions, fines, payout.
- Show your paper model. Defend one design decision.
- Live task: the facilitator proposes a new rule ("members who contribute 3 days early get a 1% discount"). You add it. On screen.

---

## One Sentence To Remember

"Model the domain. AI builds the code."
