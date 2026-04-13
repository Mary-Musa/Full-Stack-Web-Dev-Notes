# Week 20 - Day 5 Assignment

## Title
Bisect A Bug -- Git Bisect For Regression Hunting

## Overview
Week 20 Day 5 teaches `git bisect`, the fastest way to find which commit introduced a bug. You will deliberately introduce a bug in an old commit, then use bisect to walk history and find the exact commit that broke things.

## Learning Objectives Assessed
- Use `git bisect start`, `good`, and `bad`
- Navigate through history to find a regression
- Understand how binary search works in a git history
- Automate bisect with a script

## Prerequisites
- Weeks 1-19 Day 5 completed
- A repo with at least 20 commits on main

## AI Usage Rules

**Ratio:** 40/60. **Habit:** New channel handshake. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Bisect command reference.
- **NOT ALLOWED FOR:** Running bisect without understanding what it does.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Plant a bug

**What to do:**
Create a simple test that currently passes:

```javascript
// tests/math.test.js
const { add } = require("../lib/math");
test("add(2, 3) = 5", () => expect(add(2, 3)).toBe(5));
```

Make the function work correctly and commit.

Now, on a feature branch, commit 10 small unrelated changes (doc tweaks, formatting, etc).

In the middle of that branch, commit one change that breaks `add` (`return a - b` instead of `a + b`). Commit 10 more unrelated changes on top.

Merge the feature branch to main.

**Expected output:**
A history with 20+ commits, one of which broke a test that was passing.

### Task 2: Start bisect

**What to do:**
Run the test on main to confirm it is broken:

```bash
npm test
```

Then:

```bash
git bisect start
git bisect bad HEAD
git bisect good <commit-hash-before-bad-branch-started>
```

Git checks out the middle commit. Run `npm test`:
- If it passes, `git bisect good`.
- If it fails, `git bisect bad`.

Repeat until Git identifies the culprit.

**Expected output:**
Screenshot `day5-bisect.png` of Git reporting the bad commit.

### Task 3: Automate with a run script

**What to do:**
Instead of manual `good`/`bad` decisions, run bisect with a script:

```bash
git bisect start HEAD <known-good-commit>
git bisect run npm test
```

`git bisect run` automatically marks each commit good or bad based on the exit code of the script. When finished, it prints the first bad commit.

**Expected output:**
Automated bisect finds the bug without manual input.

### Task 4: Fix and clean up

**What to do:**
After bisect identifies the bad commit, run:

```bash
git bisect reset
```

This returns you to main. Now fix the bug with a new commit:

```bash
git commit -am "fix: restore add function"
```

**Expected output:**
Main builds and tests pass again.

### Task 5: Bisect notes

**What to do:**
In `bisect-notes.md`, write 5-7 sentences:
- How does bisect use binary search?
- How many steps does it take for a 128-commit history? (Hint: about 7.)
- When is automated bisect (`git bisect run`) better than manual?
- What happens if you mark a commit wrong during manual bisect?

**Expected output:**
`bisect-notes.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Use `git bisect skip` when you encounter a commit that cannot be built.
- Bisect on a real open-source project's historical bug.
- Write a CI workflow that runs bisect automatically when a test starts failing.

## Submission Requirements

- **What to submit:** Repo, screenshots, `bisect-notes.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 5

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| History with planted bug | 15 | 20+ commits, one breaks a test. |
| Manual bisect identifies the commit | 30 | Screenshot confirms. |
| Automated bisect with run | 25 | `git bisect run npm test` works. |
| Bug fixed after bisect | 10 | Main green again. |
| Bisect notes | 15 | Four questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Forgetting to `git bisect reset`.** You will be stuck in a "bisecting" state.
- **Marking a commit bad when it is good.** Undo with `git bisect log` and `git bisect replay`.
- **Running bisect on uncommitted changes.** Always start from a clean working tree.

## Resources

- Prior Day 5 assignments
- Week 20 AI boundaries: [../ai.md](../ai.md)
- Pro Git bisect: https://git-scm.com/docs/git-bisect
