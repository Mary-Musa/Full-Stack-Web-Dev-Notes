# Week 8 - Day 1 Assignment

## Title
Controlled Forms and Client-Side Validation

## Overview
Week 8 dives deeper into React. Today is all about forms: controlled inputs, multiple fields, form submission, and client-side validation. Your assignment is to build a signup form with three fields, validate them on submit, and show errors inline.

## Learning Objectives Assessed
- Build a controlled `<input>` bound to state
- Manage multiple form fields in a single state object
- Handle form submission with `event.preventDefault()`
- Validate inputs and show errors inline

## Prerequisites
- Week 7 completed

## AI Usage Rules

**Ratio this week:** 55% manual / 45% AI
**Habit:** AI as pair partner -- ask follow-ups, push back, decide yourself. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Pairing conversations to refine a validator function after you wrote the first version.
- **NOT ALLOWED FOR:** Generating your form component from scratch.
- **AUDIT REQUIRED:** Yes. Include a "Pair conversation log" section this week.

## Tasks

### Task 1: Three-field controlled form

**What to do:**
Create `src/components/SignupForm.jsx`. Requirements:

- Three fields: `name`, `email`, `password`
- All three are controlled inputs bound to state
- State stored as a single object: `const [form, setForm] = useState({ name: "", email: "", password: "" });`
- Form has a Submit button that logs the form data on submit (no backend yet)

```jsx
import { useState } from "react";

function SignupForm() {
  const [form, setForm] = useState({ name: "", email: "", password: "" });

  function updateField(field, value) {
    setForm({ ...form, [field]: value });
  }

  function handleSubmit(e) {
    e.preventDefault();
    console.log("Submitted:", form);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={form.name}
        onChange={(e) => updateField("name", e.target.value)}
        placeholder="Name"
      />
      <input
        value={form.email}
        onChange={(e) => updateField("email", e.target.value)}
        placeholder="Email"
      />
      <input
        type="password"
        value={form.password}
        onChange={(e) => updateField("password", e.target.value)}
        placeholder="Password"
      />
      <button type="submit">Sign up</button>
    </form>
  );
}

export default SignupForm;
```

Type it yourself. No AI.

**Expected output:**
Form renders, all three fields work, submit logs the form object.

### Task 2: Inline validation

**What to do:**
Add validation on submit. Before logging, check:
- `name` is at least 2 characters
- `email` contains `@` and at least one `.`
- `password` is at least 8 characters

Store errors in state:

```jsx
const [errors, setErrors] = useState({});

function handleSubmit(e) {
  e.preventDefault();
  const newErrors = {};
  if (form.name.length < 2) newErrors.name = "Name too short";
  if (!form.email.includes("@") || !form.email.includes(".")) newErrors.email = "Invalid email";
  if (form.password.length < 8) newErrors.password = "Password too short";

  if (Object.keys(newErrors).length > 0) {
    setErrors(newErrors);
    return;
  }
  setErrors({});
  console.log("Submitted:", form);
}
```

Display errors inline below each input:

```jsx
{errors.name && <p style={{ color: "red" }}>{errors.name}</p>}
```

**Expected output:**
Invalid submissions show inline errors. Valid submissions log the form object.

### Task 3: Pair conversation with AI

**What to do:**
After your validation works, start a pair conversation with AI:

- "My email validation just checks for `@` and `.`. What are the edge cases this misses?"
- Read the response. Push back: "Do I actually need to catch all of those for a signup form, or is this good enough for now?"
- Iterate. Ask follow-ups.
- Decide: either improve your validator or keep the simple version with a code comment explaining the trade-off.

Record the full pair conversation in `AI_AUDIT.md`.

**Expected output:**
Pair conversation log in the audit with at least 3 back-and-forth exchanges.

### Task 4: Disable submit until required fields are filled

**What to do:**
Disable the submit button when any required field is empty:

```jsx
const canSubmit = form.name && form.email && form.password;

<button type="submit" disabled={!canSubmit}>Sign up</button>
```

**Expected output:**
Button grey/disabled until all fields have content.

### Task 5: Write a short reflection

**What to do:**
In `day1-notes.md`, write 4-5 sentences answering:
- Why are controlled inputs "controlled"?
- What would change if you used uncontrolled inputs instead?
- One thing you learned in the pair conversation with AI.

**Expected output:**
Notes committed.

## Stretch Goals (Optional - Extra Credit)

- Add a "Show password" toggle next to the password field.
- Use `useReducer` instead of `useState` to manage the form state (preview of advanced state management).
- Write a reusable `useField(initial, validator)` custom hook.

## Submission Requirements

- **What to submit:** Repo with SignupForm, `day1-notes.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 1

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Controlled form with 3 fields | 20 | All three inputs bound to state. Submit logs the form. |
| Inline validation | 25 | Errors stored in state and rendered inline. Valid submit clears errors. |
| Pair conversation with AI | 25 | At least 3 back-and-forth exchanges. Student pushed back at least once. Decision defended. |
| Disable submit when empty | 10 | Button disables correctly. |
| Notes reflection | 10 | Three questions answered in student's own words. |
| Clean commits | 10 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Forgetting `event.preventDefault()`.** Without it, the form does a real HTTP submit and the page reloads, wiping your state.
- **Using uncontrolled inputs by accident.** If you forget `value={form.field}`, the input is uncontrolled and React will warn.
- **Accepting the first AI suggestion.** The habit is pushback. Silent acceptance is not pair programming.

## Resources

- Day 1 reading: [Forms in React.md](./Forms%20in%20React.md)
- Week 8 AI boundaries: [../ai.md](../ai.md)
- react.dev Forms: https://react.dev/learn/reacting-to-input-with-state
