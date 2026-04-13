# Week 26 - Day 5 Assignment

## Title
Phase 4 Recap And Git Hooks With Husky

## Overview
Day 5 closes Phase 4 (Weeks 20-26: automation, queues, microservices, infra). You write a reflection, ensure all CI/CD flows are green, and formalise your local pre-commit checks with Husky.

## Learning Objectives Assessed
- Reflect on a full phase of work
- Install and configure Husky hooks
- Run lint-staged on commit
- Verify end-to-end shipping works

## Prerequisites
- Days 1-4 completed

## AI Usage Rules

**Ratio this week:** 25% manual / 75% AI
**Habit:** Shipping week. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Husky boilerplate.
- **NOT ALLOWED FOR:** The reflection (that is yours).
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Install Husky and lint-staged

**What to do:**
```bash
npm install -D husky lint-staged
npx husky init
```

Edit `.husky/pre-commit`:

```bash
npx lint-staged
```

`package.json`:

```json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,yml}": ["prettier --write"]
  }
}
```

**Expected output:**
Staged files auto-formatted on commit.

### Task 2: Commit-msg hook

**What to do:**
Add `.husky/commit-msg`:

```bash
npx --no -- commitlint --edit $1
```

Install commitlint:

```bash
npm install -D @commitlint/cli @commitlint/config-conventional
echo "module.exports = { extends: ['@commitlint/config-conventional'] };" > commitlint.config.js
```

Now a commit like `fixed stuff` is rejected. `fix: the thing` is accepted.

**Expected output:**
Bad messages rejected.

### Task 3: Green-light check

**What to do:**
Confirm the full pipeline:
- Local commit -> Husky formats -> commitlint accepts
- Push -> CI lints + tests -> green
- Merge to main -> publish workflow -> GHCR image -> deploy -> smoke test passes

Screenshot each `day5-pipeline-*.png`.

**Expected output:**
Four screenshots.

### Task 4: Phase 4 reflection

**What to do:**
`PHASE_4_REFLECTION.md`, 400-500 words:
- What was Phase 4 about?
- What was the hardest topic?
- What tool/pattern did you rely on most?
- What would you do differently?
- What are you taking into the capstone?

Your own words. No AI.

**Expected output:**
Reflection committed.

### Task 5: Push everything

**What to do:**
Tag a release: `git tag -s v0.4.0 -m "Phase 4 complete"` and push. The publish workflow runs.

**Expected output:**
Signed tag, deploy visible.

## Stretch Goals (Optional - Extra Credit)

- Add a `prepare-commit-msg` hook to prepend ticket IDs.
- Configure Husky to skip on `git cherry-pick` / `git rebase`.
- Add `pre-push` to run the test suite.

## Submission Requirements

- **What to submit:** Repo, reflection, screenshots, `AI_AUDIT.md`.
- **Deadline:** End of Day 5

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Husky + lint-staged | 20 | Pre-commit formats. |
| commitlint | 15 | Bad messages rejected. |
| Full pipeline green | 25 | Four screenshots. |
| Phase 4 reflection | 30 | Substantive, honest. |
| Tagged release | 5 | Signed tag, deploy. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Bypassing hooks with `--no-verify`.** The whole point is they stop you.
- **Treating the reflection as filler.** It is the most useful file this week.
- **Forgetting to sign the tag.** You set that up in Week 25.

## Resources

- Day 5 reading: [Phase 4 Recap.md](./Phase%204%20Recap.md)
- Week 26 AI boundaries: [../ai.md](../ai.md)
- Husky: https://typicode.github.io/husky/
