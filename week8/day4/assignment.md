# Week 8 - Day 4 Assignment

## Title
Styling React -- Tailwind, Modules, and a Pre-Weekend Polish Pass

## Overview
Day 4 is pre-weekend polish day. Today you pick a styling approach for your React Task Manager and apply it consistently. You will try at least two of: plain CSS files, CSS Modules, inline styles, or Tailwind. Pick one, justify it, and refactor your components to use it.

## Learning Objectives Assessed
- Add Tailwind to a Vite + React project (or configure CSS Modules)
- Apply styling consistently across multiple components
- Make the app responsive at mobile, tablet, desktop widths
- Ship a polished, demo-ready version of your Task Manager

## Prerequisites
- Days 1-3 completed

## AI Usage Rules

**Ratio:** 55/45. **Habit:** AI as pair partner. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Asking AI to suggest Tailwind classes for a specific effect after you tried yours.
- **NOT ALLOWED FOR:** Restyling your whole app from AI output.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Pick a styling approach

**What to do:**
Read briefly about each of these options:
- Plain CSS files (import a `.css` file in your component)
- CSS Modules (`import styles from "./Button.module.css"`)
- Inline styles (`style={{ color: "red" }}`)
- Tailwind CSS (utility-first classes)

In `styling-notes.md`, write 3-5 sentences on which you picked and why. There is no wrong answer; what matters is your reasoning.

**Expected output:**
`styling-notes.md` committed.

### Task 2: Install and configure (if needed)

**What to do:**
If you picked Tailwind, install it:

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

Follow Tailwind's Vite guide to configure `content` paths and add `@tailwind` directives to your CSS.

If you picked CSS Modules, just create `.module.css` files alongside your components.

**Expected output:**
Your chosen approach is working on one test component.

### Task 3: Restyle your Task Manager

**What to do:**
Refactor your Task Manager to use the chosen approach. Requirements:
- The form inputs look polished (padding, borders, focus states)
- Completed tasks are visually distinct (strikethrough, muted colour)
- The delete button looks like a destructive action
- The container has max-width and is centered

If you use Tailwind, paste AI's suggested classes into your code only AFTER you have read what each class does.

**Expected output:**
Restyled Task Manager that looks like a real app.

### Task 4: Responsive design

**What to do:**
Make your Task Manager render well at three widths: 360px (mobile), 768px (tablet), 1280px (desktop).

If Tailwind: use the `sm:`, `md:`, `lg:` prefixes.
If plain CSS: use media queries.

Open DevTools device mode and verify at each width.

**Expected output:**
Screenshots at three widths saved as `day4-mobile.png`, `day4-tablet.png`, `day4-desktop.png`.

### Task 5: Pre-weekend checklist

**What to do:**
Create `CHECKLIST.md`:

```markdown
## Week 8 Day 4 Pre-Weekend Checklist

- [ ] Forms work with validation
- [ ] useEffect persists tasks to localStorage
- [ ] React Router has 3 routes
- [ ] Task Manager is visually polished
- [ ] Responsive at 360, 768, 1280 widths
- [ ] AI_AUDIT.md has pair conversations logged
- [ ] Repo pushed and clean
```

Tick every box honestly.

**Expected output:**
Checklist committed.

## Stretch Goals (Optional - Extra Credit)

- Add a dark mode toggle using CSS custom properties or Tailwind's `dark:` variant.
- Add a small loading skeleton that shows briefly on localStorage hydration.
- Deploy to GitHub Pages (or Netlify, or Vercel) and include the live URL.

## Submission Requirements

- **What to submit:** Repo, 3 screenshots, `styling-notes.md`, `CHECKLIST.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 4

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Styling approach chosen and justified | 15 | Notes explain the choice. |
| Approach installed and working | 15 | Configuration correct, no errors. |
| Task Manager restyled | 30 | All required elements styled (forms, buttons, completed tasks, container). |
| Responsive at three widths | 20 | Three screenshots confirm. |
| Pre-weekend checklist | 10 | All boxes honest. |
| Clean commits | 10 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Mixing approaches.** Pick one and stick with it. Mixing Tailwind with CSS Modules in the same project creates confusion.
- **Pasting AI Tailwind without understanding.** Tailwind has hundreds of classes. If you do not know what `md:grid-cols-3` means, do not ship it.
- **Forgetting focus styles.** Keyboard users rely on visible focus rings. Do not strip them out.

## Resources

- Day 4 reading: [Styling in React.md](./Styling%20in%20React.md)
- Week 8 AI boundaries: [../ai.md](../ai.md)
- Tailwind docs: https://tailwindcss.com/docs/installation/framework-guides/vite
- CSS Modules guide: https://github.com/css-modules/css-modules
