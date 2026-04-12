# Week 13, Day 4: Polishing for Feature Phones

By the end of today, the USSD app will feel solid. Error paths are handled, sessions recover from server restarts, messages fit inside the 182-character budget, strings exist in English and Kiswahili, the deepest menus are reachable in under five hops, and you have thought through how a non-literate user might navigate your menus. You will also have written a small set of unit tests against the pure state handlers, because yesterday's refactor now pays off.

This is the day most USSD tutorials skip. It is also the day that separates a demo from something Kevin's riders actually use on a cracked Nokia 105 at 6 a.m. in Kawangware.

**Prior-week concepts you will use today:**
- All of Week 13 Days 1-3 -- AT callbacks, state machines, Redis, the CRM connection
- The error handler middleware and `AppError` from Week 12 Day 2
- Node's `assert` for quick unit tests

**Estimated time:** 3 hours

---

## What We Are Polishing

Five tightly-scoped improvements:

1. **A single safety-net around the dispatcher** so the user never sees "Internal Server Error" mid-session.
2. **Swahili strings** behind a small `t()` helper so the user can pick a language on the first hop.
3. **A hard cap on screen length** enforced at render time, with sensible truncation.
4. **Session resume** when a session expires but the user redials within a short window.
5. **Unit tests for every state handler**, because they are pure functions and we can.

Nothing new under the sun -- just every one of these finished properly.

---

## 1. The Dispatcher Safety Net

Open `server/services/ussd/dispatcher.js`. Today's version needs to wrap the handler call in a try/catch:

```javascript
// server/services/ussd/dispatcher.js
const session = require("./session");
const states = require("./states");
const { t } = require("./i18n");

async function run({ sessionId, phoneNumber, rawText }) {
  try {
    const parts = rawText.split("*");
    const latestInput = rawText === "" ? "" : parts[parts.length - 1];

    const current = await session.get(sessionId);
    const lang = current.context?.lang || "en";

    const handlerName = current.state || "welcome";
    const handler = states[handlerName] || states.welcome;

    const result = await handler({
      input: latestInput,
      context: current.context,
      phoneNumber,
      lang,
    });

    if (result.response.startsWith("END")) {
      await session.destroy(sessionId);
      if (result.postSessionTask) {
        result.postSessionTask().catch((err) =>
          console.error("post-session task failed:", err)
        );
      }
    } else {
      await session.set(sessionId, {
        state: result.nextState,
        context: result.nextContext,
      });
    }

    return clampResponse(result.response);
  } catch (err) {
    console.error("ussd dispatcher error:", err);
    await session.destroy(sessionId).catch(() => {});
    return `END ${t("en", "generic_error")}`;
  }
}

function clampResponse(response) {
  // USSD budget is around 182 characters; leave 2 chars of slack.
  if (response.length <= 180) return response;
  const prefix = response.startsWith("END") ? "END " : "CON ";
  const body = response.slice(prefix.length);
  return prefix + body.slice(0, 176 - prefix.length) + "...";
}

module.exports = { run };
```

Three new things.

**The top-level try/catch** catches anything a handler throws -- a Postgres outage, a Redis hiccup, a typo in a state file -- and returns a clean `END` message to the user. Their session ends gracefully instead of showing AT's generic "Unable to process request" message.

**`session.destroy(sessionId).catch(() => {})`** on the error path swallows a secondary failure. If Redis is the thing that is down, we still want to reply to the user; we do not want the error handler to throw its own error.

**`clampResponse`** enforces the 180-character cap at the edge. Individual handlers can be sloppy with length; the dispatcher is the last line of defence. In practice the truncation rarely fires, but it prevents one bad state file from killing the whole UX.

---

## 2. Swahili Strings and i18n

USSD users in Kenya skew heavily toward Kiswahili speakers, especially in rural areas. You do not need to be a translator to support them -- you need a tiny `t()` helper and a willingness to have one state file look up strings instead of hardcoding English.

Create `server/services/ussd/i18n.js`:

```javascript
// server/services/ussd/i18n.js
const STRINGS = {
  en: {
    welcome_title: "Jetlink Support",
    welcome_menu: "1. My open tickets\n2. File a new ticket\n3. Call support\n4. Kiswahili",
    invalid: "Invalid option.",
    back: "0. Back",
    generic_error: "Something went wrong. Please try again.",
    ticket_filed: (id) => `Ticket #${id} filed. You will get an SMS shortly.`,
    no_tickets: "You have no open tickets. Dial again to file one.",
    call_support: "Call +254712000000 or dial again.",
    prompt_category: "What is the issue about?\n1. Billing\n2. Rider complaint\n3. Lost item\n4. Other",
    prompt_message: "Type a short message (3-160 chars):",
    message_too_short: "Message too short.",
    confirm_prompt: (excerpt) => `Confirm? "${excerpt}..."\n1. Yes, submit\n2. Re-type`,
  },
  sw: {
    welcome_title: "Jetlink Msaada",
    welcome_menu: "1. Tiketi zangu\n2. Ripoti tatizo\n3. Piga simu\n4. English",
    invalid: "Chaguo batili.",
    back: "0. Rudi",
    generic_error: "Tatizo limetokea. Jaribu tena.",
    ticket_filed: (id) => `Tiketi #${id} imepokelewa. Utapata SMS.`,
    no_tickets: "Huna tiketi. Piga tena kuripoti.",
    call_support: "Piga +254712000000 au jaribu tena.",
    prompt_category: "Tatizo ni kuhusu?\n1. Malipo\n2. Malalamiko ya dereva\n3. Kitu kilichopotea\n4. Mengine",
    prompt_message: "Andika ujumbe (3-160 herufi):",
    message_too_short: "Ujumbe ni mfupi sana.",
    confirm_prompt: (excerpt) => `Thibitisha? "${excerpt}..."\n1. Ndio\n2. Andika upya`,
  },
};

function t(lang, key, ...args) {
  const bundle = STRINGS[lang] || STRINGS.en;
  const value = bundle[key] || STRINGS.en[key] || key;
  return typeof value === "function" ? value(...args) : value;
}

module.exports = { t };
```

Two deliberate simplifications.

**No file-based translation files, no `i18next`, no Crowdin.** Twenty strings live in two JS objects. This is a USSD app with five screens and a four-week shelf life before it graduates to production -- a JS object is the right amount of engineering. If the app grows, promote it to JSON files later.

**Functions for strings with interpolation.** `ticket_filed` is a function that takes the id. This is cleaner than `sprintf`-style placeholders and lets the translator reorder arguments naturally (Kiswahili word order is different from English in some places).

### Using it in the welcome state

```javascript
// server/services/ussd/states/welcome.js
const { t } = require("../i18n");

module.exports = async function welcome({ input, context, lang }) {
  const currentLang = context.lang || lang || "en";

  if (input === "") {
    return {
      response: `CON ${t(currentLang, "welcome_title")}\n${t(currentLang, "welcome_menu")}`,
      nextState: "welcome",
      nextContext: { ...context, lang: currentLang },
    };
  }

  if (input === "4") {
    // Language toggle
    const newLang = currentLang === "en" ? "sw" : "en";
    return {
      response: `CON ${t(newLang, "welcome_title")}\n${t(newLang, "welcome_menu")}`,
      nextState: "welcome",
      nextContext: { ...context, lang: newLang },
    };
  }

  if (input === "1") {
    return {
      response: `CON ...`, // see my_tickets; pass-through
      nextState: "my_tickets",
      nextContext: context,
    };
  }

  if (input === "2") {
    return {
      response: `CON ${t(currentLang, "prompt_category")}\n${t(currentLang, "back")}`,
      nextState: "new_ticket_category",
      nextContext: context,
    };
  }

  if (input === "3") {
    return {
      response: `END ${t(currentLang, "call_support")}`,
      nextState: "done",
      nextContext: {},
    };
  }

  return {
    response: `CON ${t(currentLang, "invalid")}\n${t(currentLang, "welcome_title")}\n${t(currentLang, "welcome_menu")}`,
    nextState: "welcome",
    nextContext: context,
  };
};
```

The change is small: every user-facing string now goes through `t()`. The language lives in `context.lang` so it persists across hops without being inferred from the phone's locale (AT does not give us that anyway).

Apply the same pattern to every other state file. It is repetitive work -- do it once, carefully, and you never have to think about languages again.

---

## 3. Screen Length Discipline

Run this little audit script to find any state that might exceed 180 chars:

```javascript
// scripts/audit-ussd-strings.js
const STRINGS = require("../server/services/ussd/i18n").STRINGS;
// (expose STRINGS from i18n.js for this audit)

for (const [lang, bundle] of Object.entries(STRINGS)) {
  for (const [key, value] of Object.entries(bundle)) {
    if (typeof value === "string" && value.length > 120) {
      console.warn(`${lang}.${key}: ${value.length} chars`);
    }
  }
}
```

Each individual *string* should be under 120 characters so that when you combine title + menu + back option, you stay under 180. The dispatcher's `clampResponse` is a safety net, not a design target -- if your content is getting clamped in the normal flow, rewrite the content.

Rules of thumb for rewriting:

- Cut adjectives. "Please select an option" -> "Select one".
- Use the shortest verb form. "Would you like to" -> "Want to".
- Drop courtesy words. "Thank you" -> drop entirely or "Asante" (4 chars).
- Use symbols sparingly: `->` instead of "to", `+254` instead of "Kenya" ids.
- Prefer numbers in labels to prose: "5 tickets" not "You have 5 tickets".

---

## 4. Session Resume

When a user's session expires (3 minutes of inactivity or a hang-up), Redis still holds the key until the 5-minute TTL fires. If they redial within that window, they land on the welcome screen because their new session has a new `sessionId` -- and that is correct. We do not want to resume mid-flow without the user's consent. A feature-phone user redialling after a network glitch expects to start fresh; they know how to navigate back to where they were.

What we *do* want is a "continue where you left off" option at the top menu if the user had an unfinished ticket. Add a phone-based "draft" Redis key:

```javascript
// server/services/ussd/session.js
// Add:
const DRAFT_TTL = 600;
const draftKey = (phone) => `ussd:draft:${phone}`;

async function saveDraft(phone, data) {
  const client = await getClient();
  await client.set(draftKey(phone), JSON.stringify(data), { EX: DRAFT_TTL });
}

async function getDraft(phone) {
  const client = await getClient();
  const raw = await client.get(draftKey(phone));
  return raw ? JSON.parse(raw) : null;
}

async function clearDraft(phone) {
  const client = await getClient();
  await client.del(draftKey(phone));
}

module.exports = { get, set, destroy, saveDraft, getDraft, clearDraft };
```

In `new_ticket_message.js`, save the draft after the user types their message:

```javascript
await session.saveDraft(phoneNumber, {
  stage: "message_entered",
  category: context.category,
  message: truncated,
});
```

And in `new_ticket_confirm.js` when the user submits successfully:

```javascript
await session.clearDraft(phoneNumber);
```

In `welcome.js`, check for a draft on the first hit and offer a "Continue last ticket" option:

```javascript
if (input === "") {
  const draft = await session.getDraft(phoneNumber);
  const menu = draft
    ? `${t(currentLang, "welcome_menu")}\n5. Continue draft`
    : t(currentLang, "welcome_menu");
  return {
    response: `CON ${t(currentLang, "welcome_title")}\n${menu}`,
    nextState: "welcome",
    nextContext: { ...context, lang: currentLang },
  };
}
```

Handle `input === "5"` by jumping straight to `new_ticket_confirm` with the draft's context. The user sees "Confirm? [your old draft]..." and can submit or re-type.

This is a tiny feature but it is the single most appreciated UX improvement for flakey networks -- which is most networks most of the time in Kenya outside of Nairobi. A caller whose session dropped on the confirmation screen no longer has to re-type a 150-character complaint.

---

## 5. Unit Testing Pure Handlers

The whole reason we made handlers pure on Day 2 is testability. Install a tiny test runner:

```bash
npm install --save-dev node:test
```

(Node has `node:test` built in since 18. No jest, no vitest for this use case.)

Create `server/services/ussd/states/welcome.test.js`:

```javascript
// server/services/ussd/states/welcome.test.js
const { test } = require("node:test");
const assert = require("node:assert/strict");
const welcome = require("./welcome");

test("first hit returns welcome menu in english by default", async () => {
  const result = await welcome({ input: "", context: {}, phoneNumber: "+254700000000" });
  assert.ok(result.response.startsWith("CON "));
  assert.match(result.response, /Jetlink Support/);
  assert.equal(result.nextState, "welcome");
});

test("language toggle switches to kiswahili", async () => {
  const result = await welcome({
    input: "4",
    context: { lang: "en" },
    phoneNumber: "+254700000000",
  });
  assert.match(result.response, /Jetlink Msaada/);
  assert.equal(result.nextContext.lang, "sw");
});

test("invalid input re-renders menu with CON", async () => {
  const result = await welcome({ input: "9", context: {}, phoneNumber: "+254700000000" });
  assert.ok(result.response.startsWith("CON "));
  assert.match(result.response, /Invalid/);
  assert.equal(result.nextState, "welcome");
});

test("option 3 ends session with support contact", async () => {
  const result = await welcome({ input: "3", context: {}, phoneNumber: "+254700000000" });
  assert.ok(result.response.startsWith("END "));
  assert.match(result.response, /\+254712000000/);
});
```

Run with `node --test server/services/ussd/states/welcome.test.js`. You should see four passing tests.

Write the same shape of test for every state file. Focus on:

- The first-hit case (empty input).
- Each valid option.
- Invalid input.
- The back option.

You will find bugs. I promise. When writing tests for the real handlers on the real codebase you will discover at least two off-by-one errors, one missing back-button, and one place where the wrong state name is returned in an edge case. The tests catch them in milliseconds; production would catch them on a cold Monday morning with a real customer.

This is the first time in the Marathon we have written tests with any rigour. Take twenty minutes to notice how pleasant it is when the thing you are testing does not touch the database or Express. Every future test suite in this course builds on this approach: make things pure, test them.

---

## Accessibility for Feature Phones

A quick list of rules that hold for every production USSD app I have seen work well. Pin this somewhere.

1. **Numbers always mean the same thing in sibling menus.** If `0` is "back" in one sub-menu, it is "back" in every sub-menu. If `00` is "main menu", same rule. Do not make users relearn your numbers per screen.
2. **Never introduce a letter-based input.** "Type Y or N" fails on phones without a QWERTY keyboard, which is half of your audience. Always offer `1. Yes / 2. No`.
3. **Avoid dynamic option numbers.** `1. New / 2. Existing / 3. Continue draft` -- where "Continue draft" appears only sometimes -- is confusing. Some USSD apps reserve option `9` for optional features so stable options never shift. Whatever you pick, be consistent.
4. **Always show the user what they typed** in confirmation screens. `"Confirm: Valley Arcade -> Sarit"` is an opportunity to catch a mistyped category or address. Do not trust the user remembers.
5. **Keep the session linear.** Deep tree navigation confuses people on small screens. If you need depth, funnel through a clear "category first, then detail" shape.
6. **Time-to-first-screen under 500ms.** If your first response is loading user context from the database, you are already too slow. Cache the greeting or defer the lookup until a later hop.

---

## Checkpoint

1. Dispatcher catches any thrown error and returns `END <generic error>` in the user's language.
2. Dialling once, picking `4` (Kiswahili), then `2` (file a ticket) shows the category menu in Kiswahili.
3. Every response from every state is under 180 characters. Use your audit script.
4. Filing a ticket then hanging up mid-message; redialling within 10 minutes shows a "Continue draft" option at the bottom of the welcome menu.
5. `node --test server/services/ussd/states/` runs and every state has at least one test.
6. A simulated Postgres outage (stop the service) results in `END` with a friendly error, not a 500.
7. Your state machine has no string literals in English that are not in `i18n.js`. Grep to prove it.

Commit:

```bash
git add .
git commit -m "feat: polish ussd with i18n, resume, tests, error handling"
```

---

## What You Learned

Polish is not optional for channels that serve feature-phone users. A broken WhatsApp bot frustrates a Nairobi professional who opens a web dashboard instead; a broken USSD menu cuts off a farmer in Bungoma from the only tool they have. Putting in the hour for clamp, i18n, drafts, and error recovery is what turns a demo into a service.

Tomorrow is the recap, and the weekend is Project 2B -- `*384*CODE#` customer support for a real local business. The full stack you built this week (AT, Redis, state machines, Postgres, SMS) is more than enough to build something shippable in two days.
