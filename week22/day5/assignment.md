# Week 22 - Day 5 Assignment

## Title
Treasurer Dashboard And Git Rerere

## Overview
Day 5 builds a tiny treasurer dashboard (Next.js page listing cycles, contributions, and fines) and teaches `git rerere` -- the feature that remembers how you resolved a conflict so the next identical conflict resolves itself.

## Learning Objectives Assessed
- Build a read-only dashboard in a Server Component
- Enable and use `git rerere`
- Know when rerere helps and when it hurts

## Prerequisites
- Days 1-4 completed

## AI Usage Rules

**Ratio this week:** 25% manual / 75% AI
**Habit:** You model the domain yourself. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Dashboard markup, table rows.
- **NOT ALLOWED FOR:** Deciding which SQL you trust on a money screen.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Treasurer page

**What to do:**
In your Next.js app, create `app/chama/[id]/page.js` (Server Component) that queries and renders:
- Chama name and contribution amount
- The open cycle with contribution status per member (paid/pending/missing)
- Total pot so far this cycle
- Outstanding fines

Use the pg pool pattern from Week 14.

**Expected output:**
Page renders real data. Screenshot `day5-dashboard.png`.

### Task 2: Protect the page

**What to do:**
Only a logged-in treasurer should see this. Add a simple session check (reuse Week 16 session work). If not the treasurer, `notFound()`.

**Expected output:**
Logged-out users get 404.

### Task 3: Enable git rerere

**What to do:**
```bash
git config --global rerere.enabled true
```

In `rerere-notes.md`, write 5-7 sentences:
- What does rerere remember?
- How does it help when you are rebasing the same feature branch over a fast-moving `main`?
- When is rerere dangerous? (When you resolve a conflict wrong once, it repeats the wrong resolution.)
- How do you forget a bad resolution? (`git rerere forget <path>`.)

Your own words.

**Expected output:**
`rerere-notes.md` committed.

### Task 4: Trigger a conflict twice

**What to do:**
In a branch, make a change to a file on a line that also changed on main. Rebase onto main and resolve the conflict. Reset the rebase and repeat -- rerere should auto-resolve.

Record the terminal output in `rerere-demo.txt`.

**Expected output:**
`rerere-demo.txt` shows "Resolved ... using previous resolution."

### Task 5: Week 22 wrap

**What to do:**
Update the repo `README.md` with a Week 22 section summarising the Chama bot. Push everything.

**Expected output:**
README updated, Week 22 pushed.

## Stretch Goals (Optional - Extra Credit)

- Add CSV export of contributions and fines.
- Add a chart (sparkline) of contributions by cycle.
- Configure rerere in a team-shared `.gitattributes`.

## Submission Requirements

- **What to submit:** Repo, screenshot, notes, demo file, `AI_AUDIT.md`.
- **Deadline:** End of Day 5

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Treasurer page renders | 30 | Real data, correct joins. |
| Page protected | 15 | Logged-out blocked. |
| rerere enabled | 10 | Config set. |
| rerere notes | 20 | Four questions answered. |
| rerere demo | 15 | Output captured. |
| Clean commits | 10 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Dashboard showing stale data.** Force dynamic on money screens.
- **Enabling rerere after a bad resolution.** Forget the bad one first.
- **Pushing rerere-resolved commits without re-testing.** Resolution from memory is still a guess.

## Resources

- Day 5 reading: [Treasurer Dashboard.md](./Treasurer%20Dashboard.md)
- Week 22 AI boundaries: [../ai.md](../ai.md)
- git rerere: https://git-scm.com/book/en/v2/Git-Tools-Rerere
