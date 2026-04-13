# Week 8 - AI Boundaries

**Ratio this week: 55% Manual / 45% AI**
**Habit introduced: "AI as pair partner."**
**Shift from last week: React is no longer new, so the manual dip relaxes. AI becomes a conversational partner, not a passive tool.**

This week you go deeper into React: forms, side effects with `useEffect`, routing for multi-page apps, and styling. Each of these has its own gotchas, and each is a place where a good pairing partner would be useful. AI is that pair this week -- but only if you treat it like a pair, not an oracle.

A good pair partner asks questions before they answer. They say "are you sure?" when you are not. They push back when your plan is weak. This week you train yourself to talk to AI the same way, and to push back on AI in return.

---

## Why The Ratio Moved

After Week 7 the React mental model is real. You know what a component is, what state does, what props are for. That foundation is what lets you accept more AI help this week without losing the thread. You are not learning what React is anymore; you are learning how to shape it.

Forms especially benefit from AI pairing because controlled inputs are repetitive. Routing benefits because route configuration is boilerplate. Side effects benefit less -- `useEffect` is subtle and worth writing by hand for the first several examples before AI touches it.

---

## What You Will Feel This Week

- Forms will feel tedious. That is not your imagination; they are.
- `useEffect` will click on Tuesday and unclick on Wednesday. That is also normal.
- Routing will make your app feel like a real multi-page thing for the first time.
- Styling choices will eat an afternoon and feel like you learned nothing. You did -- you learned that styling choices eat afternoons.

---

## What You MUST Do Manually (55%)

### Day 1 -- Forms in React
- Write a form with three fields: text, email, password. Controlled inputs. Handle submit. Render errors below each field.
- Validate on submit first, then on blur, then on change. Compare the three UX approaches.
- One multi-step form: two pages, Next/Back navigation, data preserved between pages. All manual.

### Day 2 -- useEffect and side effects
- First five effects: write them by hand with MDN + react.dev open. Understand the dependency array.
- Effect that runs once on mount (empty dependency array).
- Effect that runs when a specific prop changes (dependency includes that prop).
- Effect with a cleanup function (subscribe/unsubscribe pattern). Return a function from your effect.
- Deliberately create a stale closure bug and fix it by adding the missing dependency.

### Day 3 -- Routing and multi-page apps
- Install React Router. Read the docs -- the v6 docs are short. Do not skip this.
- Set up three routes: `/`, `/about`, `/tasks/:id`. Render different components for each.
- Use `useNavigate` to programmatically navigate after a form submit.
- Use `useParams` to read a URL parameter inside a component.

### Day 4 -- Styling in React
- Compare three styling approaches: plain CSS files, CSS modules, inline styles. Write one component each way.
- Pick one approach for your project and stick with it. Document why in your README.
- Responsive: make your Task Manager from Week 7 look right on a 360px mobile screen.

### Day 5 and weekend -- Multi-page Task Manager
- Extend the Week 7 Task Manager into a multi-page app with routing. Pages: list, detail, new, edit.
- Form validation on add/edit with errors shown inline.
- Persist tasks in `localStorage` via a `useEffect`.

---

## You Must Break Things On Purpose

- Forget the dependency array on a `useEffect`. Observe what runs on every render.
- Add a missing dependency warning (use a prop in an effect without listing it). Let the lint catch you.
- Forget the cleanup function on a subscription-style effect. Observe the "can't update state on unmounted component" warning.
- Set up a route with a param and forget to use `useParams` -- observe the silent wrong data.

Each of these is a classic bug you will make once at work. Make it now in training.

---

## What You CAN Use AI For (45%)

Cumulative permitted uses, plus this week's new one:

1. **Explanations**
2. **Comparisons**
3. **Extending working code**
4. **Code review**
5. **Architecture sketches**
6. **Pattern scaffolding**
7. **Pair programming conversation** (new this week)

### Pair programming conversation

The new mode is: you describe a problem out loud to AI and iterate. Not one prompt, one answer. A back-and-forth where you correct AI, ask follow-ups, and push back.

Example of a real pairing conversation:

> You: "I need a form that handles three fields with validation. I am going to do it in plain React, no library. What is the simplest state shape?"
> AI: *suggests one object with all three fields*
> You: "Does that mean I call setFormData with the whole object every time any field changes?"
> AI: *confirms, suggests a helper*
> You: "That feels like a lot of re-renders. Does it matter for three fields?"
> AI: *says it does not for three, mentions it would for 30*
> You: "OK. Write me the change handler using your helper pattern."
> AI: *writes it*
> You: "That uses computed property names. Explain that syntax in one line."

Notice: you are driving. AI is answering. You ask questions AI cannot skip. You decide when the conversation ends.

### Good vs bad prompts this week

**Bad:** "Build me a form with validation."
**Good:** "I am writing a React form with three fields. I know I want controlled inputs and validation on submit. I am going to manage state as one object. Walk me through the structure -- I will implement it myself."

**Bad:** "Fix my useEffect."
**Good:** "My effect runs when I do not expect it to [paste]. I think I have a stale closure. I read my dependency array and it looks right. Can you spot what I am missing?"

**Bad:** "Which styling approach should I use?"
**Good:** "I am choosing between CSS modules and plain CSS files for a 5-component project. Give me two arguments for each. Do not recommend one -- I will decide."

---

## The Pair Partner Habit

In a real pairing session with a senior engineer, you would not accept "this is how you do it" as a final answer. You would ask "why this and not that?". This week you practice the same with AI.

Three questions to ask AI after any suggestion:

1. **"What is the alternative?"** Force it to name a different way. If it cannot, the suggestion is suspicious.
2. **"What could go wrong with this?"** Force it to admit edge cases. AI is trained to sound confident -- this question makes it slow down.
3. **"If you were reviewing this in a PR, what would you flag?"** A different frame, often yields better feedback.

Use all three questions at least once this week. Record the answers in your audit.

---

## The 25-Minute Rule

Before asking AI, spend at least twenty-five minutes of real effort. Gradient is: Week 3 was 15, Week 5 was 20, Week 8 is 25. The rule will keep slowly growing until it becomes "try until you are genuinely stuck" by Week 12.

Why the gradual climb: your own debugging capacity is growing. As you get more capable, you should need AI less often for the same kind of bug -- and when you do ask, the question should be harder, not easier.

---

## Things AI Is Bad At This Week

- **Effect dependencies.** AI is famously bad at getting the dependency array right. Use React's ESLint plugin (`react-hooks/exhaustive-deps`) and trust the linter over AI. If AI says "you don't need to add that" and the linter disagrees, the linter is right.
- **Form libraries.** AI will try to push you toward `react-hook-form` or `formik` when you want to learn forms manually. Politely decline until Week 9.
- **Routing version skew.** React Router v5 and v6 are different. AI sometimes mixes them up. Always specify the version in your prompt: "in React Router v6...".
- **Style taste.** AI has no opinion on your brand. Do not let it pick colours.

---

## Core Mental Models For This Week

- **A form is controlled state plus a submit handler.** That is the whole pattern. Everything else is variations.
- **`useEffect` is a function that runs *after* render.** It is not part of render. Changes you make inside it trigger another render.
- **The dependency array is "what makes the effect run again".** If you lie about it, your effect will have stale values.
- **A route is a mapping from URL to component.** When the URL matches, the component renders.

If something feels confusing, come back to these four.

---

## This Week's AI Audit Focus

Add a section: **Pair conversation log.** For each real pairing session (not one-shot prompts), record:

```markdown
### [Feature]
- **Opening question:** [...]
- **AI's first suggestion:** [brief]
- **My pushback:** [...]
- **AI's revised answer:** [brief]
- **Final decision:** [mine]
- **What I learned about my own taste:** [1-2 sentences]
```

At least two pair conversations must be logged this week. Facilitators grade on whether the pushback is substantive.

---

## Assessment

Week 8 assessment is a live refactor:

- Facilitator gives you a working but messy Task Manager. You have 30 minutes to improve it with one of: better state shape, routing fix, effect cleanup, or styling pass. AI allowed, but every prompt logged.
- Walk through your `useEffect` dependency array and explain each entry.
- Live change: add a new route that takes a query parameter. Implement it on screen.
- Explain what a stale closure is with an example.
- Show one pair conversation from your audit and defend your pushback.

### Live Rebuild Check

Facilitator deletes your form state management and asks you to rewrite it. If you cannot, note it and come back with proof.

---

## One Sentence To Remember

"Pair with AI the way you would pair with a senior engineer -- ask follow-ups, push back, decide." A pair session where you silently accept every suggestion is not a pair session. It is dictation.
