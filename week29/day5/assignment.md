# Week 29 - Day 5 Assignment

## Title
Week 29 Demo And Git Reflog Recovery

## Overview
Day 5 is a Week 29 checkpoint demo: integrations working end-to-end across two tenants. You also learn `git reflog` to recover from "I just did something stupid" moments.

## Learning Objectives Assessed
- Present an integrations demo to a non-technical viewer
- Use `git reflog` to recover lost commits
- Know when reflog helps vs when it cannot

## Prerequisites
- Days 1-4 completed

## AI Usage Rules

**Ratio this week:** 25% manual / 75% AI
**Habit:** Integrations week. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Demo script.
- **NOT ALLOWED FOR:** The demo reflection.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Two-tenant demo

**What to do:**
Record a 5-minute screen demo that:
- Shows Tenant A signup, USSD flow, WhatsApp flow, M-Pesa pay
- Shows Tenant B doing the same with different credentials
- Shows the inbox for A does not contain B's messages
- Shows the wallet for A is not B's

Save as `demo-week29.md` (script) and/or `demo-week29.mp4`.

**Expected output:**
Committed.

### Task 2: Demo script

**What to do:**
`capstone/demo-script.md`: five sections, minute-by-minute, of what to say while showing each step. Your future self will thank you at the Week 30 capstone demo.

**Expected output:**
Committed.

### Task 3: git reflog practice

**What to do:**
Simulate a disaster:

```bash
git checkout -b throwaway
git reset --hard HEAD~3
# oh no
git reflog
git reset --hard HEAD@{1}
```

Record the session in `reflog-demo.txt`.

**Expected output:**
Committed.

### Task 4: Reflog notes

**What to do:**
`reflog-notes.md`, 5-7 sentences:
- What does reflog store? (Local only, ref movements.)
- How long are entries kept by default? (90 days for reachable, 30 for unreachable.)
- What does reflog NOT save you from? (Files never committed; stash never saved; remote force-push.)
- What command finds a "dangling" commit? (`git fsck --lost-found`.)

Your own words.

**Expected output:**
Committed.

### Task 5: Week 29 wrap

**What to do:**
Tag `v0.29.0`, push, update the capstone README with Week 29 section.

**Expected output:**
Pushed.

## Stretch Goals (Optional - Extra Credit)

- Write a `scripts/save-reflog.sh` that copies reflog to a file before risky operations.
- Try `git fsck --lost-found` on a demo repo.
- Set `gc.reflogExpire` to a longer value.

## Submission Requirements

- **What to submit:** Repo, demo, script, reflog demo, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 5

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Two-tenant demo | 30 | All four flows. |
| Demo script | 15 | Five sections. |
| Reflog practice | 20 | Recovered. |
| Reflog notes | 25 | Four questions answered. |
| Tag + README | 5 | Pushed. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Demo without a script.** Nerves kill it.
- **Relying on reflog for uncommitted work.** It does not help.
- **Forcing reflog cleanup with `git gc --prune=now`.** You lose recovery.

## Resources

- Day 5 reading: [Week 29 Demo.md](./Week%2029%20Demo.md)
- Week 29 AI boundaries: [../ai.md](../ai.md)
- git reflog: https://git-scm.com/docs/git-reflog
