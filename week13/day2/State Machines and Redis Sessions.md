# Week 13, Day 2: State Machines and Redis Sessions

By the end of today, the hello menu from Day 1 will be a proper multi-step USSD app with a real state machine, Redis holding session state, input validation, back navigation, and the ability to hold structured data (like a ride request) across screens. The if/else chain from yesterday will be gone, replaced by a tiny engine that reads the user's current state, runs a handler, and decides what state to go to next.

This is the day USSD stops being a demo and starts being a tool you can ship. Every real USSD app in Kenya -- M-Pesa menus, bank balance checks, school fee checks, Safaricom's `*144#`, Equitel -- is a state machine at the core. Today you learn to build one.

**Prior-week concepts you will use today:**
- The USSD CON/END protocol and the breadcrumb `text` field (Week 13, Day 1)
- Express routing (Week 10)
- Node modules and exports (Week 2)
- The repository pattern (Week 12, Day 2) -- you will use the same habit for "state handlers"

**Estimated time:** 4 hours

---

## Why The Breadcrumb Approach Falls Apart

Yesterday's handler was an `if (text === "1") { ... } else if (text === "1*2") { ... }` chain. For three menus with one option each, that works. Try to build a ride-booking flow with it:

1. Welcome menu: `1. Book ride / 2. Cancel ride / 3. Support`
2. User picks 1. Show pickup location options: `1. Current area / 2. Yaya / 3. Valley Arcade`
3. User picks 2. Show dropoff location options: `1. Sarit / 2. Westgate / 3. Junction`
4. User picks 1. Confirm: `1. Yes / 2. No`
5. User picks 1. Book the ride.

The breadcrumb on step 5 is `1*2*1*1`. To handle that, your code has to:

```javascript
if (text === "1*2*1*1") {
  // what does 1 mean at position 0? "Book ride"
  // what does 2 mean at position 1? "Yaya"
  // what does 1 mean at position 2? "Sarit"
  // what does 1 mean at position 3? "Confirm yes"
  // ... book the ride
}
```

You are reverse-engineering meaning from positions. Change the order of anything, break every branch. Add a "back" option, break everything again. Store a typed name in the middle of the flow ("please type your name"), and now `text` is `"1*2*Wanjiru*1"` -- a name disguised as a menu option. Users type stars in names? Good luck.

This is the point where people quit and decide USSD is "primitive". It is not. It just needs a better mental model.

### The right model: state plus context

A USSD session has two things:

- **A current state**, a short string naming where the user is in the flow. Examples: `welcome`, `choose_pickup`, `choose_dropoff`, `confirm_booking`, `done`.
- **A context**, a small object of typed values the user has given us so far. Example: `{ pickup: "Yaya", dropoff: "Sarit" }`.

On every POST from AT, the server:

1. Loads the `(state, context)` for this `sessionId` from Redis.
2. Runs the handler for the current state with the user's latest input.
3. Handler returns: the response text, the next state, and the updated context.
4. Server saves the new state and context to Redis, replies to AT.

If you squint, the handler looks exactly like an Express controller -- input in, decision made, output out. Stateful, but in the sense that the state lives in Redis and the handler itself is pure.

---

## Installing Redis

Redis is an in-memory key-value store. Think of it as a giant JavaScript object that survives process restarts and can be shared between multiple Node processes. USSD sessions are the textbook use case: small, short-lived, read/written on every hop, must be fast.

### Ubuntu/Debian/WSL

```bash
sudo apt install redis-server
sudo systemctl enable --now redis-server
```

### Mac

```bash
brew install redis
brew services start redis
```

### Windows

Use WSL. Native Windows Redis exists but it is a pain.

### Verify

```bash
redis-cli ping
# PONG
```

If you see `PONG`, Redis is up. Do a couple of commands to get a feel:

```bash
redis-cli
127.0.0.1:6379> SET mykey "hello"
OK
127.0.0.1:6379> GET mykey
"hello"
127.0.0.1:6379> EXPIRE mykey 60
(integer) 1
127.0.0.1:6379> TTL mykey
(integer) 58
127.0.0.1:6379> DEL mykey
(integer) 1
127.0.0.1:6379> exit
```

Five commands and you have learned 80% of what you need from Redis today: `SET`, `GET`, `EXPIRE`, `TTL`, `DEL`. Key lives until you delete it or its TTL runs out. We will use a 300-second TTL on USSD sessions because AT's own session is three minutes max.

---

## Node + Redis

Install the official client:

```bash
npm install redis
```

Create `config/redis.js`:

```javascript
// config/redis.js
const { createClient } = require("redis");

const client = createClient({
  url: process.env.REDIS_URL || "redis://localhost:6379",
});

client.on("error", (err) => {
  console.error("Redis error:", err);
});

let connected = false;

async function getClient() {
  if (!connected) {
    await client.connect();
    connected = true;
  }
  return client;
}

module.exports = { getClient };
```

The wrapper exists so that every file that needs Redis calls `getClient()` and gets the same shared connection. Connecting multiple times would work -- node-redis handles it gracefully -- but one shared client is cleaner.

---

## The Session Store

Create `sessionStore.js`:

```javascript
// sessionStore.js
const { getClient } = require("./config/redis");

const SESSION_TTL = 300; // 5 minutes -- a bit more than the USSD max.

function key(sessionId) {
  return `ussd:session:${sessionId}`;
}

async function get(sessionId) {
  const client = await getClient();
  const raw = await client.get(key(sessionId));
  if (!raw) {
    return { state: "welcome", context: {} };
  }
  return JSON.parse(raw);
}

async function set(sessionId, data) {
  const client = await getClient();
  await client.set(key(sessionId), JSON.stringify(data), { EX: SESSION_TTL });
}

async function destroy(sessionId) {
  const client = await getClient();
  await client.del(key(sessionId));
}

module.exports = { get, set, destroy };
```

Three functions, no surprises. `get` initialises a fresh session if there is nothing in Redis yet (first hit of the session), `set` writes with an expiry, `destroy` clears state on `END`.

Key naming matters. `ussd:session:<sessionId>` is a namespace -- later when Redis holds other things (BullMQ queues in Week 23, cached rider locations for Week 20), a prefix keeps them from colliding. Get the habit now.

---

## The State Machine

The core idea: states are just named handler functions, each handler is pure (input in, output out), and one dispatcher picks the right handler.

Create `states/index.js`:

```javascript
// states/index.js
const welcome = require("./welcome");
const choosePickup = require("./choosePickup");
const chooseDropoff = require("./chooseDropoff");
const confirmBooking = require("./confirmBooking");

module.exports = {
  welcome,
  choose_pickup: choosePickup,
  choose_dropoff: chooseDropoff,
  confirm_booking: confirmBooking,
};
```

Now each state file is a single function that knows nothing about Express, Redis, or AT. Create `states/welcome.js`:

```javascript
// states/welcome.js
module.exports = function welcome(input, context) {
  if (input === "") {
    return {
      response: "CON Jetlink Boda\n1. Book a ride\n2. My rides\n3. Support",
      nextState: "welcome",
      nextContext: context,
    };
  }

  if (input === "1") {
    return {
      response:
        "CON Pick up from?\n1. Yaya\n2. Valley Arcade\n3. Sarit\n0. Back",
      nextState: "choose_pickup",
      nextContext: context,
    };
  }

  if (input === "2") {
    return {
      response: "END Your rides: (coming on Day 3)",
      nextState: "done",
      nextContext: context,
    };
  }

  if (input === "3") {
    return {
      response: "END Call +254712000000 for support.",
      nextState: "done",
      nextContext: context,
    };
  }

  return {
    response: "CON Invalid option.\n1. Book a ride\n2. My rides\n3. Support",
    nextState: "welcome",
    nextContext: context,
  };
};
```

Notice the contract every handler follows:

- **Input:** the user's most recent answer (just the last `text` segment), and the current context object.
- **Output:** `{ response, nextState, nextContext }`.
- The handler never reads or writes Redis. It never touches `req`/`res`. You could unit-test it with a plain `expect(welcome("1", {})).toEqual(...)` -- no HTTP, no database, no simulator.

This purity is the whole reason for the refactor. Every state file is a small, fast, testable piece. The dispatcher wires them together.

Create `states/choosePickup.js`:

```javascript
// states/choosePickup.js
const LOCATIONS = { "1": "Yaya", "2": "Valley Arcade", "3": "Sarit" };

module.exports = function choosePickup(input, context) {
  if (input === "0") {
    return {
      response: "CON Jetlink Boda\n1. Book a ride\n2. My rides\n3. Support",
      nextState: "welcome",
      nextContext: {},
    };
  }

  const pickup = LOCATIONS[input];
  if (!pickup) {
    return {
      response:
        "CON Invalid. Pick up from?\n1. Yaya\n2. Valley Arcade\n3. Sarit\n0. Back",
      nextState: "choose_pickup",
      nextContext: context,
    };
  }

  return {
    response:
      "CON Drop off at?\n1. Westgate\n2. Junction\n3. Sarit\n0. Back",
    nextState: "choose_dropoff",
    nextContext: { ...context, pickup },
  };
};
```

Notice the two patterns worth memorising.

**Back goes to `welcome` with an empty context.** Going back resets any half-entered data, which is what users expect -- they typed the wrong thing and want to start over.

**Invalid input re-renders the same menu in `CON`**, not `END`. A bad keystroke should not close the session. This is the most common junior mistake and feature-phone users hate it.

Create `states/chooseDropoff.js`:

```javascript
// states/chooseDropoff.js
const LOCATIONS = { "1": "Westgate", "2": "Junction", "3": "Sarit" };

module.exports = function chooseDropoff(input, context) {
  if (input === "0") {
    return {
      response:
        "CON Pick up from?\n1. Yaya\n2. Valley Arcade\n3. Sarit\n0. Back",
      nextState: "choose_pickup",
      nextContext: { ...context, pickup: undefined },
    };
  }

  const dropoff = LOCATIONS[input];
  if (!dropoff) {
    return {
      response:
        "CON Invalid. Drop off at?\n1. Westgate\n2. Junction\n3. Sarit\n0. Back",
      nextState: "choose_dropoff",
      nextContext: context,
    };
  }

  const { pickup } = context;
  return {
    response: `CON Confirm ${pickup} -> ${dropoff}?\n1. Yes\n2. No`,
    nextState: "confirm_booking",
    nextContext: { ...context, dropoff },
  };
};
```

Notice the confirmation screen uses the stored pickup from `context` -- that is the payoff of holding state in Redis instead of parsing from `text`.

Create `states/confirmBooking.js`:

```javascript
// states/confirmBooking.js
module.exports = function confirmBooking(input, context) {
  if (input === "1") {
    // On Day 3 we will actually create a ride row and send notifications.
    return {
      response: `END Ride booked: ${context.pickup} -> ${context.dropoff}. You will receive an SMS shortly.`,
      nextState: "done",
      nextContext: {},
    };
  }

  if (input === "2") {
    return {
      response: "END Cancelled. Dial again to try anew.",
      nextState: "done",
      nextContext: {},
    };
  }

  return {
    response: `CON Invalid. Confirm ${context.pickup} -> ${context.dropoff}?\n1. Yes\n2. No`,
    nextState: "confirm_booking",
    nextContext: context,
  };
};
```

Four files, one per state, each a single function. Clean.

---

## The Dispatcher

Now the glue. Replace `index.js`:

```javascript
// index.js
require("dotenv").config();
const express = require("express");
const session = require("./sessionStore");
const states = require("./states");

const app = express();
app.use(express.urlencoded({ extended: false }));

app.post("/ussd", async (req, res) => {
  const { sessionId, text } = req.body;

  const parts = text.split("*");
  const latestInput = text === "" ? "" : parts[parts.length - 1];

  const { state, context } = await session.get(sessionId);
  const handler = states[state] || states.welcome;

  const { response, nextState, nextContext } = handler(latestInput, context);

  if (response.startsWith("END")) {
    await session.destroy(sessionId);
  } else {
    await session.set(sessionId, { state: nextState, context: nextContext });
  }

  res.set("Content-Type", "text/plain");
  res.send(response);
});

app.listen(process.env.PORT || 5001, () => {
  console.log("USSD server running");
});
```

Thirty lines. Read what it does:

1. Pull the *latest* answer from the breadcrumb -- not the whole history, just the tip.
2. Load state and context from Redis.
3. Pick the handler for the current state.
4. Run it.
5. If the handler ended the session, destroy the Redis entry. Otherwise save the new state.
6. Reply.

The only reason we still read `text` at all is to get the user's most recent input. The earlier parts are ignored because we are tracking navigation through Redis, not through AT's breadcrumb. This is a big shift from Day 1 and it is the whole point of state machines.

---

## Test It

Run the server. Fire up the AT simulator. Dial `*384*1234#`. You should see the welcome menu. Pick 1, pick 2 (Valley Arcade), pick 1 (Westgate), pick 1 (yes). You should see:

```
Ride booked: Valley Arcade -> Westgate. You will receive an SMS shortly.
```

Now dial again and try the back flow. Pick 1 (book a ride), pick 2 (Valley Arcade), pick 0 (back). You should be back on the pickup menu with your previous pickup cleared. Pick 0 again and you are back at the root menu. Pick 2 and you see "My rides (coming on Day 3)".

Watch the server logs. For each hop you should see one database-free, Redis-only request. Check the Redis CLI while a session is live:

```bash
redis-cli
127.0.0.1:6379> KEYS ussd:session:*
1) "ussd:session:ATUid_abc123"
127.0.0.1:6379> GET ussd:session:ATUid_abc123
"{\"state\":\"choose_dropoff\",\"context\":{\"pickup\":\"Valley Arcade\"}}"
127.0.0.1:6379> TTL ussd:session:ATUid_abc123
(integer) 287
```

That is the whole magic. State is a key, context is JSON, TTL is a clock.

---

## Typed Input, Not Just Numbers

Numbers-only menus cover most USSD apps, but sometimes you need the user to type something -- a name, a PIN, an amount. The state machine handles this trivially. Add a new state, `enter_pin`:

```javascript
// states/enterPin.js
module.exports = function enterPin(input, context) {
  if (!/^\d{4}$/.test(input)) {
    return {
      response: "CON PIN must be 4 digits. Enter your PIN:",
      nextState: "enter_pin",
      nextContext: context,
    };
  }
  return {
    response: "CON PIN accepted.\n1. Continue",
    nextState: "pin_ok",
    nextContext: { ...context, pin: input },
  };
};
```

Two things worth noting:

**Validation is re-rendering, not crashing.** When the PIN is wrong, the handler returns `CON` with a re-prompt and stays in the same state. The user tries again. The session stays alive.

**Never store PINs in plain text in Redis.** This state is fine for a demo, but if this were real, you would hash the PIN with bcrypt before putting it in context. Redis is memory, but memory gets dumped, logged, and stolen. Treat it like any other data store.

For today's ride-booking flow we do not need typed input. The pattern is here for tomorrow.

---

## A Word On Session Lifetime

The TTL on our Redis key is 300 seconds. The AT session lifetime is 180 seconds on most carriers. Why the mismatch?

Because if your server crashes and restarts, and the user redials right away, you want them to land on a fresh session, not a half-dead one. The TTL is the garbage collector. 5 minutes is plenty.

Also: when we call `session.destroy()` on an `END` response, we explicitly clear the key even though it would have expired naturally. Why? Because the user might immediately redial. A fresh dial should not resume the dead session. Explicit `DEL` is the safe move.

---

## Checkpoint

1. `redis-cli ping` returns `PONG`.
2. Your Day 1 if/else chain is gone. `index.js` is under 40 lines and contains no menu text.
3. You have four state files, one per screen, each a pure function.
4. Dialling the code and going through the full happy path books a ride and ends the session.
5. Pressing `0` from any sub-menu returns you to the parent menu. Pressing `0` from the root menu is handled (it is the "invalid" branch).
6. Typing something invalid at a menu re-renders the same menu with `CON`, not `END`. Verify for each state.
7. While a session is mid-flow, `redis-cli KEYS "ussd:session:*"` shows the key and `GET` shows the JSON context.
8. Five minutes after the last hop, `TTL` goes to `-2` (key is gone).
9. You can write one more state (say, `support_submenu`) and plug it in by editing only `states/index.js` and creating the file. No other file changes.

Commit:

```bash
git add .
git commit -m "feat: state machine with redis session store"
```

---

## What You Learned

- A USSD session is `(state, context)` in Redis, not a breadcrumb string.
- Handlers are pure functions. Express and Redis are thin wrappers.
- Validation re-prompts; it does not end sessions.
- Redis has TTLs -- use them so stale state cleans itself up.

Tomorrow we stop using hardcoded locations and connect USSD to the Week 12 CRM database. Customers will look themselves up by phone number, see their real rides, request new ones that create real database rows, and get real SMS confirmations through Africa's Talking.
