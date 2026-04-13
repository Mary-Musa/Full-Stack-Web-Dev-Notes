# Week 6 - Day 5 Assignment

## Title
Ship the Data Dashboard With a Proper Release and a Clean PR

## Overview
Week 6 Day 5 combines your Git skills (branches, PRs, tags, rebases) with the Data Dashboard project. Today you ship the dashboard via a proper feature-branch PR, rebase it onto main, tag a v0.2.0 release, and publish it via GitHub Pages so the link is shareable.

## Learning Objectives Assessed
- Ship a real feature via a PR, not by pushing to main
- Rebase a feature branch before merging
- Tag and publish a release
- Deploy a static site via GitHub Pages
- Write a meaningful release description

## Prerequisites
- Week 4 Day 5 (stash, cherry-pick, reflog) and Week 5 Day 5 (tags, rebases) completed
- Day 4 assignment: dashboard skeleton working locally

## AI Usage Rules

**Ratio:** 65/35. **Habit:** MDN first. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Debugging a GitHub Pages 404 after you read the Pages docs.
- **NOT ALLOWED FOR:** Generating workflow YAML for GitHub Pages without reading the docs.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Ship the dashboard on a feature branch

**What to do:**
1. From main, create `feature/data-dashboard`.
2. Flesh out the dashboard to at least THREE widgets, each fetching a different piece of data from one or more public APIs.
3. Each widget handles loading, success, and error states.
4. Commit at least 4 times on this branch with conventional messages.

**Expected output:**
Feature branch with multiple commits, pushed to GitHub.

### Task 2: Open a PR with a template

**What to do:**
Open a PR from `feature/data-dashboard` to `main`. Use the What / Why / How to test template. In the description include:
- Screenshot of the dashboard
- Link to the public API(s) you used
- One sentence on how you handle rate limits (if applicable)

**Expected output:**
PR open on the repo.

### Task 3: Rebase the feature branch before merging

**What to do:**
Before merging, simulate someone else having merged to main:
1. On main, make one unrelated commit (edit README, commit, push).
2. On your feature branch, rebase onto the updated main:

```bash
git fetch origin
git rebase origin/main
```

If conflicts occur, resolve them manually. No AI. Push the rebased branch with `git push --force-with-lease`.

4. On the PR page, GitHub will show the updated, linear history.

**Expected output:**
Screenshot `day5-rebased-pr.png` showing the PR with the rebased commits.

### Task 4: Merge and tag a release

**What to do:**
1. Merge the PR on GitHub.
2. Switch back to main locally and pull.
3. Tag the current main commit as v0.2.0:

```bash
git checkout main
git pull
git tag -a v0.2.0 -m "Week 6: Data Dashboard shipped"
git push origin v0.2.0
```

4. On GitHub, publish a release from the tag with a short description.

**Expected output:**
Release visible on GitHub.

### Task 5: Deploy via GitHub Pages

**What to do:**
1. In your repo settings, find the "Pages" section.
2. Under "Build and deployment", pick "Deploy from a branch" -> `main` -> `/` (root).
3. Save. Within a minute or two, GitHub Pages will show your dashboard live at `https://YOUR_USERNAME.github.io/REPO_NAME/`.
4. Add the live URL to your repo's README.

**Expected output:**
Live URL visible to the public. Screenshot of the deployed page as `day5-live.png`.

## Stretch Goals (Optional - Extra Credit)

- Use a GitHub Actions workflow to deploy on every push to main.
- Add a custom domain to your Pages site.
- Include a README badge showing the build status.

## Submission Requirements

- **What to submit:** Repo link, PR link, release link, live URL, three screenshots.
- **Deadline:** End of Day 5

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Feature branch with multiple commits | 15 | 4+ conventional commits on the branch. |
| PR with template | 15 | Proper What/Why/How to test sections. Screenshot included. |
| Rebase before merging | 20 | Branch rebased cleanly. Linear history visible. Force-push-with-lease used. |
| Merge + tag + release | 20 | PR merged. v0.2.0 tag pushed. GitHub release published with description. |
| GitHub Pages live | 20 | Dashboard accessible at a public URL. Link in README. |
| AI Audit | 5 | MDN links included. |
| Clean commits | 5 | Conventional messages throughout. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **`git push --force` instead of `--force-with-lease`.** `--force-with-lease` refuses to overwrite remote commits you have not seen. Safer.
- **Deploying before testing locally.** Your local dashboard works, but GitHub Pages is case-sensitive on file names and absolute paths. Test with relative paths.
- **Leaving API keys in the deployed code.** Public Pages sites expose everything. If you accidentally committed a key, rotate it immediately.

## Resources

- Prior Day 5 assignments (Weeks 1-5)
- Week 6 AI boundaries: [../ai.md](../ai.md)
- GitHub Pages docs: https://docs.github.com/en/pages
- Git rebase docs: https://git-scm.com/docs/git-rebase
