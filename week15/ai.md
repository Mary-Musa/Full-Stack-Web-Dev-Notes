# Week 15 - AI Boundaries

**Ratio this week: 35% Manual / 65% AI**
**Habit introduced: "Model the state machine on paper."**
**Shift from last week: 5% more AI, and a new habit specifically for flow-heavy weeks.**

This week you build the cart and checkout state machine for the Next.js shop. State machines are a pattern you met in Week 13 (USSD) -- now you apply the same pattern in a different context. That repetition is the reason the ratio loosens: you already know the shape. What you do not know yet is the e-commerce-specific state transitions and the Server Actions layer.

The habit is narrow and specific: before you write any state-machine code, draw the full diagram on paper. States, transitions, guards, and the data each state carries. Every programmer eventually develops this habit. You are developing it on schedule.

---

## What You Will Feel This Week

- The first time `useReducer` clicks and feels cleaner than five `useState`s, you will be annoyed you did not use it sooner.
- Server Actions will feel like magic for mutations -- "I just call a function and the server handles it?"
- The state machine will have one edge case you did not draw on paper, and that edge case will bite you on Friday.

---

## What You MUST Do Manually (35%)

### Day 1 -- The cart state machine
- **Draw it first.** On real paper or a whiteboard. States: `empty`, `populated`, `checking_out`, `confirmed`. Transitions: `add_item`, `remove_item`, `clear`, `proceed_to_checkout`. No code yet.
- Translate the drawing into a `useReducer` hook by hand. One reducer, one set of actions. Test each action in isolation.
- Persist the cart to `localStorage` via a custom hook.

### Day 2 -- The checkout state machine
- Draw the checkout machine on paper: `info`, `payment`, `confirmed`, `error`. What data does each state carry? What triggers each transition?
- Write it in code. Name each action clearly.

### Day 3 -- Server Actions and form mutations
- Read the Next.js docs on Server Actions. Not an AI summary.
- Write one Server Action by hand that adds a product to the cart. Call it from a form.
- Write a second Server Action that handles checkout submission. Handle the redirect after success.

### Day 4 -- Orders and admin
- Build a minimal admin view that lists orders.
- Hook up the order detail page using dynamic routes from Week 14.

---

## What You CAN Use AI For (65%)

- **Cart UI scaffolding.** Repeated pattern.
- **Admin dashboard.** Repeated pattern.
- **Cleaning up the reducer** after the first version works.
- **Generating Tailwind styles** for the checkout form.

AI is forbidden for:
- Drawing the state machine.
- Deciding on the action names.
- Picking which data each state carries.
- Handling the money/payment side (payments dip is next week).

### Good vs bad prompts this week

**Bad:** "Build me a cart in Next.js."
**Good:** "Here is my cart reducer [paste]. I realised I have no handler for 'update quantity'. What is the smallest new action and case I need to add?"

**Bad:** "Write my checkout flow."
**Good:** "I drew my checkout machine on paper: info -> payment -> confirmed. Here is the diagram as text [paste]. Help me translate it into a useReducer shape. I will add the UI myself."

---

## The Paper-First Habit

Every state-machine week this programme has (USSD, cart, checkout, chama cycles, capstone multi-tenant lifecycle) gets a paper drawing first. Photograph it. Include it in the audit. Show it in assessment.

Why: state machines are hard to change after they are coded, and easy to change on paper. An hour with a pen saves eight hours with a keyboard.

---

## The 25-Minute Rule

Same. State bugs take longer to debug than syntax bugs -- be patient.

---

## Things AI Is Bad At This Week

- **Off-by-one in reducers.** Classic `removeItem` bug: the index is off. Verify with tests.
- **Stale cart in Server Actions.** AI sometimes writes code that reads from stale client cache. Always refetch after a mutation.
- **Redirect after success.** Next.js has a specific `redirect()` helper. AI sometimes uses `router.push` in a Server Action, which is wrong.

---

## Core Mental Models For This Week

- **A state machine is a map of allowed transitions.** If a transition is not in the map, it cannot happen.
- **A reducer is a pure function: `(state, action) -> newState`.** No side effects inside.
- **Server Actions run on the server and can read secrets and write to the DB.** Client calls them like regular async functions.

---

## This Week's AI Audit Focus

Attach a photo (or ASCII sketch) of your paper state machine diagram. No diagram = no passing this audit.

---

## Assessment

- Show your paper diagram. Walk the facilitator through every state and transition.
- Live task: add a new state to your cart (e.g., `abandoned` after 30 minutes of inactivity). Update the reducer on screen.
- Explain why you chose `useReducer` over multiple `useState`s.

---

## One Sentence To Remember

"Paper before pixels, always for state machines."
