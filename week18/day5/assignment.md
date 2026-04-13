# Week 18 - Day 5 Assignment

## Title
Security Scanning With Dependabot And A CodeQL Workflow

## Overview
Week 18 Day 5 adds automated security scans to your GitHub repo. Dependabot watches your dependencies for known vulnerabilities; CodeQL statically analyses your code for patterns like SQL injection or XSS. Both are free for public repos.

## Learning Objectives Assessed
- Enable Dependabot alerts on a GitHub repo
- Configure a Dependabot config for automated PRs
- Add a CodeQL workflow for static analysis
- Triage and fix (or dismiss) a security alert

## Prerequisites
- Weeks 1-17 Day 5 completed

## AI Usage Rules

**Ratio:** 45/55. **Habit:** Audit every security line. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Config YAML generation.
- **NOT ALLOWED FOR:** Dismissing alerts without reading what they mean.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Enable Dependabot alerts

**What to do:**
1. Go to your repo on GitHub -> Settings -> Code security and analysis.
2. Enable: Dependabot alerts, Dependabot security updates, Dependabot version updates.

Verify Dependabot alerts are active.

**Expected output:**
Screenshot `day5-dependabot-enabled.png`.

### Task 2: Configure Dependabot version updates

**What to do:**
Create `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/packages/backend"
    schedule:
      interval: "weekly"
  - package-ecosystem: "npm"
    directory: "/packages/frontend"
    schedule:
      interval: "weekly"
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "monthly"
```

Adjust paths to your actual package layout.

Commit and push. Dependabot starts scanning.

**Expected output:**
After a few minutes, Dependabot opens PRs for outdated dependencies.

### Task 3: Review a Dependabot PR

**What to do:**
Pick one Dependabot PR. Read what it is updating. Merge it (if safe) or dismiss it (if you disagree).

Write down your reasoning in `dependabot-notes.md`:
- Which package was updated?
- From what version to what version?
- Was the changelog scary?
- Did you merge or dismiss, and why?

**Expected output:**
`dependabot-notes.md` committed.

### Task 4: Add CodeQL workflow

**What to do:**
Create `.github/workflows/codeql.yml`:

```yaml
name: CodeQL
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 0 * * 0" # weekly

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    strategy:
      matrix:
        language: [javascript]
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
      - uses: github/codeql-action/analyze@v3
```

Push. Wait for the workflow to run.

**Expected output:**
CodeQL workflow runs. Security tab shows results (even if empty).

### Task 5: Triage a finding (or its absence)

**What to do:**
If CodeQL finds an issue, read the description in the Security tab. Fix it or write a comment explaining why it is a false positive.

If it finds nothing, write a note in `dependabot-notes.md` explaining that this week's code is clean and listing three patterns you would look for manually.

**Expected output:**
Notes updated.

## Stretch Goals (Optional - Extra Credit)

- Add a secrets-scanning alert for committed secrets.
- Add `gitleaks` as a pre-commit hook to catch secrets locally.
- Set up GitHub's branch protection to require the CodeQL check before merging.

## Submission Requirements

- **What to submit:** Repo with Dependabot config, CodeQL workflow, `dependabot-notes.md`, screenshots.
- **Deadline:** End of Day 5

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Dependabot alerts enabled | 15 | Screenshot confirms. |
| dependabot.yml committed | 20 | Correct schedule and directory paths. |
| Real Dependabot PR reviewed | 25 | Decision documented. |
| CodeQL workflow runs | 25 | Workflow green or findings documented. |
| Notes | 10 | Reasoning documented. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Merging every Dependabot PR blindly.** Major version bumps can break your app. Read the changelog.
- **Ignoring every alert.** Some are real. Triage them.
- **Committing to main without waiting for CodeQL.** Require the check in branch protection if you can.

## Resources

- Prior Day 5 assignments
- Week 18 AI boundaries: [../ai.md](../ai.md)
- Dependabot docs: https://docs.github.com/en/code-security/dependabot
- CodeQL docs: https://codeql.github.com/
