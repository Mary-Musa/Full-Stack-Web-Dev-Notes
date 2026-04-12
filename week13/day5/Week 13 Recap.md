# Week 13, Day 5: Week Recap and Phase 2 Close-Out

This week closed out the entire communication stack. You can now reach Kenyan users through WhatsApp, through a web dashboard, and through USSD -- the three channels every local product needs. Phase 2 of the Marathon is complete. From Monday onward we move into Phase 3: commerce, Next.js, and payments.

**Estimated time:** 90 minutes of review, 90 minutes of peer pairing.

---

## The Four Shifts This Week

### 1. From "I don't get USSD" to "USSD is an HTTP POST"

All the telco mystique around USSD collapses once you see the one thing it actually is: Africa's Talking POSTs form data to your URL, you reply with plain text that starts with `CON` or `END`, the user's phone shows it. Everything else is UX and state management on top of that.

### 2. From `text` breadcrumbs to state machines in Redis

The naive `if (text === "1*2*1")` approach works for three-option menus and breaks for everything real. Day 2's refactor gave you:

- `(state, context)` in Redis, keyed by session id.
- Pure handler functions per state.
- A dispatcher that is ~30 lines.
- Testable handlers.

You will use exactly this pattern for Telegram bots in Week 20, for the WhatsApp bot when we revisit it in Week 22, and -- implicitly -- for anywhere in the Marathon that a user walks through a multi-step flow.

### 3. From standalone demo to one CRM, three channels

Day 3's merge folded USSD into the Week 12 Postgres CRM. Now a dispatcher on the web dashboard sees tickets from WhatsApp and USSD side by side. A `channel` column and a tiny `tickets.repo.js` is all the schema work that was needed. This is how unification actually looks: not a rewrite, but a careful addition with clear boundaries.

### 4. From English-only English to bilingual, polished, error-proofed

Day 4 took the thing from working to reliable. `t()` helper, two-language strings, length clamp, session drafts, top-level error catch, unit tests. Polish is not optional for feature phone users.

---

## Phase 2 Close-Out Checklist

Phase 2 of the Marathon roadmap was titled "Data, Logic & CRM Engineering". Every bullet is now shipped:

| Roadmap bullet | Where |
|---|---|
| MongoDB & Mongoose (schema design, relationships) | Week 12 Day 4 (conceptual -- Postgres hands-on instead) |
| JWT auth, password hashing, middleware protections | Week 12 Day 3 |
| API architecture: controllers, services, error handling | Week 12 Day 2 |
| Relational data: PostgreSQL & SQL | Week 12 Days 1-2 |
| WhatsApp Business API: webhooks, templates, 24-hour windows | Week 11 Days 1-4 |
| USSD engineering: state management with Redis (session hops) | Week 13 Days 1-4 |
| Project 2A: WhatsApp Lead Capture CRM | Week 11 weekend |
| Project 2B: USSD Customer Support | Week 13 weekend |

Phase 2 is done. Phase 3 starts Monday.

---

## Self-Review Questions

Answer these out loud before the peer session.

**Protocol**

1. What do `CON` and `END` mean, and what happens on the next hop for each?
2. Why does AT POST with `application/x-www-form-urlencoded` instead of JSON?
3. What four fields does AT put on every USSD POST, and which one holds the user's history so far?
4. What is the character budget per screen, and what happens if you exceed it?
5. How long does a USSD session live before the carrier kills it?

**State management**

6. What is in a Redis session value, and why is that richer than the `text` breadcrumb alone?
7. What TTL do we set on USSD session keys and why is it longer than the carrier's session lifetime?
8. What is the shape of a "pure state handler" in this codebase? What are its inputs and outputs?
9. Where does the "back" button live in the data model, and what resets when the user presses it?
10. How does the dispatcher know which handler to run for an incoming hop?

**Integration**

11. Why do we fold USSD into the existing `leads` table with a `channel` column instead of creating a new `tickets` table?
12. Why is the SMS confirmation sent in a `postSessionTask` instead of before the `END` response?
13. What would happen to the user's session if Redis went down mid-flow, and how does the dispatcher handle it?
14. On USSD, who is the authentication authority, and what do you rely on from them?
15. If you wanted to add a PIN step before a money-moving action, where would the state file go and how would you validate the input?

**Polish**

16. Why is every string run through `t(lang, key)` instead of hardcoded?
17. What does `clampResponse` protect you against and when does it actually fire?
18. Why can you unit-test state handlers without running Redis or Postgres?
19. Name three accessibility rules for feature-phone UX and why each matters.
20. What is a "session draft" and which user problem does it solve?

Target: 16/20 clear answers. Anything below, go back to the corresponding day.

---

## Peer Coding Session (Friday, 3 hours)

Pair up and pick one track.

### Track A: Deep menu -- "track my package"

Build a new top-level option: `3. Track shipment`. The flow is:

1. Prompt for tracking number (validate: 8 chars, alphanumeric).
2. Look up in a `shipments` table you create (`id`, `tracking_code`, `status`, `current_location`, `eta`).
3. Show the latest status, location, and ETA.
4. Offer `1. Refresh / 2. Contact sender / 0. Back`.

Success criteria: the new option lives in its own state files, does not require changes to existing states, and has unit tests. This proves the architecture scales.

### Track B: Analytics dashboard for USSD

On the web dashboard, add a `/ussd-stats` page (admin only) showing:
- Sessions today, this week, this month.
- Top 5 categories of tickets filed via USSD.
- Average session length (how many hops before `END`).
- Drop-off rate (sessions that ended without reaching the final screen).

This requires one new table (`ussd_events`) that the dispatcher writes to on every hop, plus one SQL query per stat. Bonus: put the counters behind a Redis `INCR` for speed.

### Track C: Pairwise happy-path testing

Without talking to each other, each of you dials the code ten times and tries to break it. Then compare notes. Every time one of you finds a weird bug (session gets stuck, menu renders wrong, SMS does not send, duplicate tickets filed) the other has to write a failing test before fixing it. Target: 3 bugs found, 3 tests added.

### Track D: Rewrite for Airtel Money quirks

The AT sandbox is generous. Real Safaricom is less so. Read the AT docs about "MNO timeouts" and identify three quirks that would bite you in production:
- Safaricom drops sessions after 90s of no activity.
- Airtel has tighter latency budgets.
- Telkom's session ids can collide across carriers in rare cases.

Propose one code change for each quirk and pair-review the proposals. No code required, just PR-style write-ups.

---

## Preparing for the Weekend

This weekend you ship Project 2B -- dial-in customer support for a real local business. Read the brief in `week13/weekend/project.md` before you leave today and sketch the menu tree on paper. Three questions to answer in your head tonight:

1. Which SME are you targeting? (Pick one. Jetlink Boda is fine; so is your cousin's salon, your neighbour's kibanda, or any real place you can dial.)
2. What are the top three things that SME gets asked on the phone every day? Those are your top-level menu options.
3. How will the SME owner see and action the tickets that come in? (Reuse the Week 12 dashboard, or sketch one new page.)

Come to the peer session with a menu tree on paper or in a text file. You will refine it with your partner, then build it Saturday and Sunday.

Phase 2 is done. Next week we move to Next.js -- a step change in how you build the frontend. You will miss Express-only weeks by Wednesday.
