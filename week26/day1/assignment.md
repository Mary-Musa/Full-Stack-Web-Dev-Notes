# Week 26 - Day 1 Assignment

## Title
GitHub Actions CI -- Lint, Test, Build

## Overview
Week 26 is CI/CD plus two mini-projects. Today you wire GitHub Actions to lint, test, and build your code on every push and PR.

## Learning Objectives Assessed
- Write a GitHub Actions workflow
- Cache node_modules for faster runs
- Run tests against a real Postgres service
- Require CI green before merging

## Prerequisites
- Weeks 1-25 completed

## AI Usage Rules

**Ratio this week:** 25% manual / 75% AI
**Habit:** Shipping week -- let AI do the heavy lifting, you verify. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Workflow YAML, matrix setup.
- **NOT ALLOWED FOR:** The branch protection rules (that is policy).
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Lint + test workflow

**What to do:**
`.github/workflows/ci.yml`:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready --health-interval 5s
          --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm test
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/test
```

**Expected output:**
Green check on GitHub.

### Task 2: Build matrix

**What to do:**
Add `strategy.matrix.node: [18, 20]` to test against two Node versions. Keep it small -- matrices cost CI minutes.

**Expected output:**
Two jobs run in parallel.

### Task 3: Cache

**What to do:**
The `cache: npm` above hits npm's cache directory. Verify the second run is faster in Actions logs.

**Expected output:**
Second run logs show cache hit.

### Task 4: Branch protection

**What to do:**
In GitHub Settings > Branches > main > Require a pull request, require CI checks. Save.

Try pushing directly to main from a branch -- GitHub refuses.

**Expected output:**
Screenshot `day1-protected.png`.

### Task 5: Notes

**What to do:**
In `ci-notes.md`, write 5-7 sentences:
- Why is running tests in CI more trustworthy than on your laptop?
- What does "required status check" do?
- Why run tests on PRs AND on main? (Catching issues after merge.)
- What is a matrix for? When is it overkill?

Your own words.

**Expected output:**
`ci-notes.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Add concurrency cancellation for superseded runs.
- Upload coverage to Codecov.
- Run a lint-only job on docs folders.

## Submission Requirements

- **What to submit:** Repo, workflow, screenshot, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 1

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| CI workflow | 30 | Lint + test green. |
| Matrix | 10 | Two Node versions. |
| Caching | 10 | Second run faster. |
| Branch protection | 25 | Required check visible. |
| CI notes | 20 | Four questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Tests depending on env secrets in PRs from forks.** Secrets are not exposed to fork PRs by default.
- **Forgetting `cache: npm`.** Runs are 3x slower.
- **No branch protection.** All the CI effort is optional if anyone can force push.

## Resources

- Day 1 reading: [GitHub Actions CI.md](./GitHub%20Actions%20CI.md)
- Week 26 AI boundaries: [../ai.md](../ai.md)
