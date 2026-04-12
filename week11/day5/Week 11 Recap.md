# Week 11, Day 5: Recap -- What You Actually Learned

This week did not just teach you "how to build a WhatsApp bot." It gave you a collection of patterns that show up in almost every real-world backend you will write from now on. Today we pause, look back at what those patterns are, and make sure they are actually in your head -- not just on your screen.

There is no new code today. Instead, we walk through every concept the week introduced, ask you to answer a few questions about each without looking at the tutorial files, and point at which day to revisit if anything feels fuzzy. At the end, there is a short self-check -- ten questions. If you can answer all ten without scrolling back, you are ready for the weekend project. If you cannot, reread the day the weak spot came from before you start building.

**Estimated time:** 1-2 hours of active review.

---

## The Project In One Paragraph

You built a WhatsApp Lead Capture CRM. Customers message a WhatsApp number, a state machine bot collects their name, email, and inquiry type through a short conversation, and the details land in a SQLite database as a structured lead. A React + Tailwind dashboard reads those leads, lets a sales team search and filter them, open a slide-out panel with the full conversation history, and move them through a pipeline (new -- contacted -- qualified -- converted -- lost) with one click. New leads appear on the dashboard within ten seconds without a page refresh.

That is the elevator pitch. Everything else in this recap is the skills that got you there.

---

## The Five Patterns That Matter Most

### 1. Webhooks

A **webhook** is a URL on *your* server that a third party calls when an event happens. It is the reverse of a normal API call. Instead of you calling Meta to ask "any new messages?", Meta calls you the instant a message arrives.

This week you built two halves of a webhook:

- **The verification handshake** (GET request). One-time. Meta sends `hub.mode`, `hub.verify_token`, and `hub.challenge` as query parameters; if the token matches your `.env`, you echo the challenge back as plain text. Day 1.
- **The real event handler** (POST request). Every time a user sends a message. You parse the JSON body, do your work, and return 200. Day 2.

You already saw this shape last week with the M-Pesa callback from Safaricom. The vocabulary ("webhook", "verify token", "callback URL") changes between APIs but the shape does not. When you learn Stripe webhooks, Slack webhooks, GitHub webhooks, or Twilio webhooks next, you are applying the same pattern to a different vendor.

**Question to answer without looking:** Why does a webhook server need to return 200 *before* it finishes its work?

### 2. State Machines For Conversations

An HTTP server forgets everything between requests. A conversation has memory. The bridge between the two is a **state machine**: a small set of states, stored in the database, that decides what the next incoming message *means*.

Your bot had five states (`awaiting_name`, `awaiting_email`, `awaiting_inquiry_type`, `confirming`, `complete`) and one escape hatch (`restart`). Every webhook call: load the current state from SQLite, run the matching `switch` branch, save the new state, send a reply.

The central insight: **the code is just the transitions; the database holds the state.** If you ever catch yourself reaching for a module-level variable to "remember" something about a user, stop and put it in the database instead.

This pattern generalises far beyond chat bots. Checkout flows, onboarding wizards, multi-step forms, and approval workflows are all state machines under the hood.

**Question:** Why can't you put the conversation state in a module-level `const userStates = {}` object inside `bot.js`?

### 3. HMAC Signature Verification

A public webhook URL is reachable by anyone on the internet. Nothing stops a malicious person from POSTing fake "messages" to your server and filling your CRM with junk. HMAC signatures are how you prove a request actually came from the vendor.

Meta computes `HMAC-SHA256(requestBody, yourAppSecret)` and puts it in the `X-Hub-Signature-256` header. You compute the same thing on your end and compare. If they match, the request is real; if they do not, you reject with 401.

Two things made this work:

1. The `verify` callback on `express.json()` that stashes the raw request bytes on `req.rawBody` *before* parsing. Without the raw bytes, you cannot reproduce the hash.
2. `crypto.timingSafeEqual` instead of `===` to compare the two hashes. Always use timing-safe comparison for secrets.

**Question:** Why does `express.json()` without the `verify` option break HMAC verification?

### 4. The REST + Partial Update Pattern

On Day 3 you wrote four route handlers that between them cover most of what a typical CRUD backend ever does:

| Route | What It Teaches |
|---|---|
| `GET /api/leads` | Dynamic WHERE-clause building, pagination with `LIMIT`/`OFFSET`, search with `LIKE`, returning `total` alongside the page. |
| `GET /api/leads/:id` | Joining related data (lead + conversation + messages) into one response so the frontend makes one call, not three. |
| `PATCH /api/leads/:id` | Partial updates with a dynamic `SET` clause, server-side enum validation, 404 vs 400 distinction, `updated_at` timestamp bumping. |
| `GET /api/stats` | Aggregate queries with `COUNT(*)` and `GROUP BY`, zero-filling missing rows so the frontend never gets `undefined`. |

If you can reproduce those four shapes for any resource (leads, products, invoices, users), you can build the backend for most CRUD apps.

**Question:** Why PATCH instead of PUT for status updates from the dashboard?

### 5. Polling + `refreshKey`

The dashboard refetches on a 10-second interval so new leads appear without a page reload. The pattern:

```javascript
const [refreshKey, setRefreshKey] = useState(0);

useEffect(() => { fetchLeads(); }, [refreshKey, search, status]);

useEffect(() => {
  const id = setInterval(() => setRefreshKey(k => k + 1), 10000);
  return () => clearInterval(id);
}, []);
```

One state variable. Any action that should trigger a refetch -- filter change, polling tick, child component finishing a PATCH -- bumps `refreshKey` (or is a dependency of the effect directly). The cleanup function on the interval effect is mandatory, not optional -- without it you leak timers across hot reloads.

This is the simplest "make the screen feel live" trick in React. Upgrade to SSE or WebSockets only when polling becomes visibly laggy.

**Question:** Why does the polling effect have an empty `[]` dependency array while the fetch effect depends on `refreshKey`?

---

## Day-By-Day Concept Map

Use this to find the day to revisit if a topic feels weak.

### Day 1 -- Foundations

- Two-folder layout (`server/`, `client/`) reused from Week 10.
- `.env.example` discipline: commit an empty template, never the real secrets.
- SQLite schema for a CRM: `leads`, `conversations`, `messages`. Why `wa_phone` is `UNIQUE`, why `state` lives in its own table, why we log raw payloads.
- The `express.json({ verify })` raw-body hook (for Day 2's signature check).
- Meta Developer App setup: app, phone number ID, access token, app secret, verify token.
- ngrok as an HTTPS tunnel for localhost (same as M-Pesa callback setup from Week 10).
- Webhook verification handshake: GET with `hub.mode`, `hub.verify_token`, `hub.challenge`.

### Day 2 -- The Bot

- `services/whatsapp.js` with `sendText` and `sendInquiryList` wrapping Meta's Graph API via `axios`.
- Interactive list messages (tap-to-choose) vs plain text; the 24-hour customer service window.
- The state machine pattern in a `switch` statement.
- Validation as conversation: `looksLikeEmail`, `looksLikeName`, re-ask on invalid input.
- Edge cases: voice notes, images, emoji, `restart` command, repeat messages after `complete`.
- Immediate `res.sendStatus(200)` before doing work, then `try/catch` around the work.
- HMAC-SHA256 signature verification with `crypto.createHmac` and `crypto.timingSafeEqual`.

### Day 3 -- The REST API

- Lead lifecycle: new, contacted, qualified, converted, lost. Why statuses are enforced in the route layer.
- Dynamic WHERE clause building: parallel `where` and `params` arrays joined with ` AND `.
- Pagination: `Math.max(1, ...)`, `Math.min(100, ...)`, `LIMIT`/`OFFSET`, returning `total`.
- Full-text-ish search with `LIKE '%term%'` across three columns.
- The detail route joining lead + messages + conversation state in one JSON response.
- PATCH with dynamic `SET` clauses, checking `!== undefined` (not falsy), always bumping `updated_at`.
- Stats with `COUNT(*)`, `GROUP BY status`, and zero-fill for missing statuses.

### Day 4 -- The Dashboard

- Tailwind v3 install for CRA: `tailwindcss@3 postcss autoprefixer`, `tailwind.config.js`, three `@tailwind` directives in `index.css`.
- The API service layer (`services/api.js`) with `axios.create({ baseURL })`.
- `StatsCards` with `refreshKey` for polling-driven refetch.
- `LeadsTable` with the `STATUS_STYLES` map for colour-coded badges; SQLite date string normalisation (`replace(" ", "T") + "Z"`).
- `LeadDetail` slide-out with two `useEffect`s (fetch + Escape key), async `changeStatus`, `wa.me/<phone>` direct reply link, inbound/outbound chat bubble layout.
- `Dashboard` page state (`search`, `status`, `leads`, `loading`, `error`, `selectedId`, `refreshKey`) and the two effects that drive fetching and polling.
- CORS rule: `app.use(cors())` must come before route registrations.

---

## What You Are *Not* Expected To Remember

You do not need to memorise Meta's exact JSON payload shape -- you will always look it up. You do not need to remember every Tailwind class -- you will always search the docs. You do not need to type `crypto.createHmac` from scratch -- you will paste it from the docs or a prior project.

What you *do* need to remember:

1. The *shape* of the solution -- "this is a webhook problem, I need a verify handler and a POST handler".
2. Which pattern matches which problem -- "this is a multi-turn thing, I need a state machine".
3. Where to look for the exact code -- "I wrote this on Day 3 of Week 11, go look".

Memorisation is not the skill. Pattern recognition is.

---

## Self-Check (Answer Without Scrolling Back)

Write your answers on paper or in a scratch file. Do not open the tutorial files until you have attempted all ten.

1. What three things does Meta give you on the **API Setup** page that you must put in your `.env` file?
2. Why does the webhook POST handler call `res.sendStatus(200)` *before* running the bot logic instead of after?
3. In one sentence, why is the conversation state stored in the `conversations` table instead of in a JavaScript variable?
4. Give two reasons our `GET /api/leads` route validates `status` against a `VALID_STATUSES` array instead of trusting whatever the client sent.
5. Why do we check `status !== undefined` in the PATCH route instead of `if (status)`?
6. Explain what the `refreshKey` state variable on the dashboard actually does. Why is there *one* key shared between polling, filter changes, and the detail panel's `onUpdated` callback?
7. What goes wrong if you remove the `verify` option from `app.use(express.json(...))` in `index.js`?
8. What happens in the bot if the user sends a voice note while the conversation is in state `awaiting_email`?
9. Why does `GET /api/stats` zero-fill the `byStatus` object in JavaScript instead of relying on the SQL `GROUP BY` result?
10. What Tailwind utility classes would you use to build a full-screen overlay with a dim background and a right-aligned panel? (You do not need exact class names -- just describe the approach.)

### Answer Check

If any of these tripped you up, revisit the day listed next to it before starting the weekend project.

| # | Topic | Revisit |
|---|---|---|
| 1 | Meta config | Day 1 |
| 2 | Webhook ack discipline | Day 2 |
| 3 | State machine persistence | Day 2 |
| 4 | Input validation / defence in depth | Day 3 |
| 5 | Truthy vs undefined in PATCH | Day 3 |
| 6 | Polling + `refreshKey` | Day 4 |
| 7 | Raw body for HMAC | Day 1 setup / Day 2 verify |
| 8 | Non-text message handling | Day 2 |
| 9 | Zero-fill aggregation | Day 3 |
| 10 | Tailwind slide-out layout | Day 4 |

---

## Before You Start The Weekend Project

Three practical things to do before you open the weekend brief:

1. **Clean up your `whatsapp-crm` repo.** Run `git status`. Commit anything outstanding. Make sure your `.env` is in `.gitignore` and has never been committed -- run `git log --all --oneline -- .env` to confirm it returns nothing.
2. **Freeze this week's work.** Tag the current commit as `week11-complete`: `git tag week11-complete && git push --tags`. That way if you break something during the weekend project you always have a known-good checkpoint to return to.
3. **Rotate your Meta access token if you committed it anywhere.** Temporary tokens expire after 24 hours anyway, but if the token ever made it into a commit, delete the app on Meta's dashboard and create a fresh one. Secrets that touch git history are burned forever.

When those are done, open the weekend project in `week11/weekend/`.
