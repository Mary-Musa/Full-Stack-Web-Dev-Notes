# Week 19 - Day 5 Assignment

## Title
Semantic Release And Automated Publishing Via CI

## Overview
Week 19 Day 5 automates what you did by hand on Day 4. You will install `semantic-release` (or similar), wire it to GitHub Actions, and have every merge to `main` trigger a version bump and auto-publish -- with zero manual commands.

## Learning Objectives Assessed
- Install and configure semantic-release
- Configure GitHub Actions to run semantic-release on main
- Use an NPM_TOKEN secret for auth
- Watch a merge automatically result in a published version

## Prerequisites
- Weeks 1-18 Day 5 completed

## AI Usage Rules

**Ratio:** 35/65. **Habit:** You design the API. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Workflow YAML boilerplate.
- **NOT ALLOWED FOR:** Pasting configs without understanding what they do to your repo.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Install semantic-release

**What to do:**
```bash
cd packages/mctaba-payments
npm install -D semantic-release @semantic-release/git @semantic-release/changelog
```

Add `release.config.js`:

```javascript
module.exports = {
  branches: ["main"],
  plugins: [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    "@semantic-release/npm",
    "@semantic-release/git",
    "@semantic-release/github",
  ],
};
```

**Expected output:**
Config committed.

### Task 2: NPM token secret

**What to do:**
1. Go to npmjs.com > Account > Access Tokens > Generate New Token (Automation).
2. Copy the token.
3. In your GitHub repo > Settings > Secrets and variables > Actions > New repository secret.
4. Name: `NPM_TOKEN`. Value: the token.

**Expected output:**
Secret set.

### Task 3: Release workflow

**What to do:**
Create `.github/workflows/release.yml`:

```yaml
name: Release
on:
  push:
    branches: [main]
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
      pull-requests: write
      packages: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
        working-directory: packages/mctaba-payments
      - run: npx semantic-release
        working-directory: packages/mctaba-payments
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

**Expected output:**
Workflow committed.

### Task 4: Trigger a release via PR

**What to do:**
Create a feature branch. Make a real change (e.g., add a feature or fix a bug). Commit with a conventional message (`feat:` or `fix:`). Open a PR. Merge it.

semantic-release analyses the commits, bumps the version, updates CHANGELOG, publishes to npm, and creates a GitHub release -- all automatically.

**Expected output:**
Screenshot `day5-auto-release.png` of the workflow run and the new version on npm.

### Task 5: Notes on automation

**What to do:**
In `automation-notes.md`, write 5-7 sentences:
- What problem does semantic-release solve compared to manual versioning?
- Why do you need `fetch-depth: 0` in the checkout step?
- What secrets does the workflow need and why?
- When would you NOT use this (hint: private packages with paid customers who prefer manual releases)?

Your own words.

**Expected output:**
`automation-notes.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Configure different release channels (alpha, beta, stable) using `branches`.
- Set up a preview-release workflow for PRs using `prerelease-github-action`.
- Add Slack notification after every release.

## Submission Requirements

- **What to submit:** Repo with release config, workflow, screenshots, `automation-notes.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 5

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| semantic-release config | 15 | release.config.js committed. |
| NPM_TOKEN secret added | 10 | Secret present (not committed). |
| release.yml workflow | 25 | Runs on main. |
| Real auto-published version | 30 | New version visible on npm as a result of a merge. |
| Automation notes | 15 | Four questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Forgetting `fetch-depth: 0`.** Without the full git history, semantic-release cannot analyse commits.
- **Manual version bumps that fight the tool.** Let semantic-release own the version.
- **Not understanding the commit message rules.** Only `feat:`, `fix:`, and `BREAKING CHANGE` footer cause releases. `chore:` and others do not.

## Resources

- Prior Day 5 assignments
- Week 19 AI boundaries: [../ai.md](../ai.md)
- semantic-release: https://semantic-release.gitbook.io/semantic-release/
