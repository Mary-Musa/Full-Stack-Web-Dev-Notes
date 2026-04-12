# Week 13 Weekend Project: Dial *384*CODE# Customer Support

Pick a real local business. Build a USSD customer-support line for it using everything from Week 13 and the CRM from Weeks 11-12. By Monday morning your chosen SME should have a working number they can put on a poster outside their shop. This is **Project 2B** on the Marathon roadmap.

**Estimated time:** 10-12 hours, split across Saturday and Sunday.

**Deadline:** Monday morning before Week 14 Day 1.

---

## Pick a Business

This is not a fictional project. Pick a real place, preferably one you can walk to. Examples:

- A hair salon that books appointments by phone and loses calls every day.
- A boda stage with a shared phone number and no tracking.
- A sacco that answers the same "what is my balance?" question 200 times a week.
- A school tuck shop that takes lunch orders from parents.
- Your cousin's mitumba stand that runs on WhatsApp and still misses orders.

Talk to the owner. Ask them what three to five questions customers ask most. Ask them how they currently handle each. Ask them what they would like to automate. Twenty minutes of conversation is worth more than any design exercise you could do alone. If you cannot reach a real business, pick Jetlink Boda from the week's examples -- but prefer real.

Write their top five questions down. These are your top-level menu options. Some examples of what you might hear:

- **Salon:** "Is X available on Friday?" / "How much is a hair treatment?" / "Is the shop open now?" / "Do you do braids?" / "Can I book an appointment?"
- **Boda stage:** "Where is my rider?" / "What is the fare to X?" / "Are you online?" / "Report rider issue" / "Lost something on a ride"
- **Sacco:** "My balance?" / "My last deposit?" / "Fees due?" / "Contact manager"

---

## Functional Requirements

### 1. Top-level menu

- At most 5 options.
- Must include a "Contact support" option that ends the session with a phone number or short instruction.
- Must include a language toggle (English / Kiswahili) somewhere.
- Must fit in 180 characters.

### 2. At least three functional sub-menus

Each one is a real feature, not a placeholder. For a salon, that might be:
1. **Check opening hours** (immediate `END` with today's hours, pulled from Postgres)
2. **Book an appointment** (multi-step: service, date, time, confirm)
3. **Ask a question** (free text, becomes a lead in the CRM)

Every sub-menu must:
- Handle invalid input with a `CON` re-prompt (never `END` on a typo).
- Offer a `0. Back` option.
- Fit in 180 characters per screen.
- Be covered by at least one unit test.

### 3. Real data, real writes

- At least one option must read from Postgres (e.g., "check opening hours" pulls from a `business_hours` table).
- At least one option must write to Postgres (e.g., "book appointment" creates an `appointments` row).
- At least one option must send an SMS confirmation via Africa's Talking (e.g., "your appointment is confirmed for Friday 3pm").

### 4. Dashboard integration

The SME owner needs a way to see what came in. Either:
- Reuse the Week 12 dashboard (filter by `channel = 'ussd'` and add your new category filters), **or**
- Add one new page to the dashboard that shows the specific entity you are creating (e.g., "Appointments" for the salon).

Whichever you pick, the owner must be able to log in, see incoming items, and take one action on them (e.g., confirm the appointment, mark the question answered).

### 5. Bilingual

Every user-facing string must go through `t(lang, key)`. English and Kiswahili, both complete. No mixed-language screens.

### 6. Polished

- Length clamp in the dispatcher.
- Top-level error catch returns a graceful `END`.
- Draft resume on at least one multi-step flow.
- Unit tests for at least 60% of your state files.

---

## Technical Requirements

1. Code lives in the same `server/` project as Weeks 11-12, under `services/ussd/`.
2. Schema changes go in `db/schema.sql` and load cleanly on a fresh database.
3. A `db/seed.sql` creates: one admin user, the business's reference data (hours, services, fare table, whatever), and three sample records (one existing appointment, one old ticket, one lead).
4. A `README.md` section describes:
   - Which business you picked and the five questions you asked them.
   - The menu tree (in text or ASCII art).
   - How to test each flow.
   - What you would add next if you had another week.
5. Africa's Talking API key is in `.env`, not in git.

---

## Grading Rubric (100 pts)

| Area | Points | What earns it |
|---|---|---|
| Real business fit | 10 | The five questions came from a real conversation; the menu matches what customers actually ask. |
| Menu tree + state machine | 15 | All states are pure functions; dispatcher is <40 lines; menu tree is sensible. |
| Sub-menus | 15 | Three functional flows; each one works end to end. |
| Database reads/writes | 15 | Real schema changes; real rows get created; queries are indexed. |
| SMS confirmation | 10 | Fires via AT, the recipient gets it, errors are handled gracefully. |
| Dashboard integration | 10 | Owner can log in and see the incoming items; one action works. |
| Bilingual strings | 10 | Every string via `t()`; Kiswahili is reasonable, not machine-translated gibberish. |
| Polish | 10 | Length clamp, error catch, draft resume, unit tests. |
| README | 5 | Clear, includes the business story and menu tree. |

Anything under 60 points gets a rewrite request.

---

## Submission Checklist (demo to your peer Monday morning)

- [ ] Dial the code from the AT simulator. Welcome menu appears in English.
- [ ] Pick the language toggle. Menu re-renders in Kiswahili.
- [ ] Walk through your three sub-menus end to end.
- [ ] The action that creates a database row does so; show the row in `psql`.
- [ ] The action that sends SMS does so; show the log line or the received SMS.
- [ ] Log in to the dashboard as the SME owner, see the item you just created, take one action on it.
- [ ] Force an error (stop Postgres, disconnect Redis). The session ends gracefully.
- [ ] Run the test suite. It passes.
- [ ] Tell the story in two sentences: "I built X for Y because customers kept asking Z."

---

## Hints

**On picking a business.** The best projects for this weekend come from SMEs who are drowning in phone calls. If the owner says "I spend two hours a day on the phone answering the same question", that is your flow. If they say "I am doing fine", pick another one -- you will not be motivated to finish by Sunday night.

**On menu tree design.** Draw it on paper before you write any code. The tree should fit on one A4 sheet. If it does not, it is too deep -- cut branches until it fits. You will be surprised how much you can accomplish with 3 levels and 4 options per level (4^3 = 64 leaf states).

**On `channel` in the dashboard.** If you reuse the `leads` table, set `channel = 'ussd'` and use `category` to distinguish your sub-flows ("appointment", "question", "complaint"). If you create a new table (e.g., `appointments`), join on `lead_id` so you can still tie everything back to a customer phone number.

**On SMS sandbox.** AT's sandbox SMS does not charge you, but it also only sends to sandbox-simulator numbers by default. To send to your real phone, go to the sandbox settings and add your phone to the "simulator contacts". Otherwise the send will look like it worked in the logs but nothing arrives.

**On the owner's dashboard.** Do not build a new frontend from scratch. Add one route to the Week 12 React dashboard. The bar is low -- a list, a detail view, one button. The goal is proving the data is queryable, not shipping a new product.

**On the Kiswahili strings.** Use Kiswahili sanifu (standard), not Sheng. Feature-phone users skew toward standard Swahili. If you are not a Kiswahili speaker, ask one student in the cohort to review your strings for 10 minutes. No one will grade you on translation quality but do not ship Google Translate output.

**On what NOT to do.** Do not redesign the menu Saturday night after you already built it. Do not try to build a new dashboard from scratch. Do not integrate a new payment provider (we do that in Week 17). Do not add authentication to USSD beyond the phone-number-is-identity pattern. Do not install a new framework. Focus.

---

## If You Finish Early

- **Add appointment reminders via cron.** A simple 5-minute cron job that queries appointments for the next 24 hours and sends an SMS reminder. (Week 21 teaches cron formally, but `node-cron` is one npm install away.)
- **Voice fall-through.** AT supports voice calls too. Add a single option `9. Call an agent` that triggers an outbound call instead of ending the session.
- **Multi-tenant from day one.** Make the welcome screen branded per SME, with the SME id stored in a config table keyed by the service code. If you rent multiple USSD codes from AT, the same server can host multiple businesses.

Good luck. Dial in, build something real, and come back Monday with a story. Week 14 starts Next.js and things get cinematic.
