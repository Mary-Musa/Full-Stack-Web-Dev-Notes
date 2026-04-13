# Week 26 - AI Boundaries

**Ratio this week: 25% Manual / 75% AI**
**Habit introduced: "AI scaffolds, you review the pipeline."**
**Shift from last week: Another 5% to AI. CI/CD YAML is pure template territory.**

You add GitHub Actions for CI/CD and ship Projects 5 (Booking System) and 6 (Notification Hub). The CI/CD side is template work; Projects 5 and 6 are compositions of patterns you have built many times.

The habit is the same shape as Week 25's: AI scaffolds the pipeline, you own the verification. Pipelines that "work" can still be dangerously wrong -- deploying to production when they should deploy to staging, skipping tests that should run, storing secrets incorrectly.

---

## What You MUST Do Manually (25%)

### Day 1 -- GitHub Actions CI
- Read the GitHub Actions docs intro. Understand workflows, jobs, steps.
- Write your first workflow by hand: checkout, install, test. Trigger on push.
- Verify the workflow runs green on a commit.

### Day 2 -- Docker publishing and deploys
- Push your Docker image to a registry (GitHub Container Registry, Docker Hub).
- Wire up a deploy step: on push to `main`, build, test, publish.
- Protect: no deploys to production from feature branches.

### Day 3 -- Project 5 (Booking System)
- Reuse: Postgres schema, Express API, React dashboard, notifications. Almost nothing new.
- Manual only: the booking state machine (you know the drill by now -- state machines are always drawn on paper first).

### Day 4 -- Project 6 (Notification Hub)
- Consolidate all notification channels (WhatsApp, Telegram, SMS) into one hub.
- Reuse: queue from Week 23, channels from previous weeks.
- Manual only: the routing logic that decides which channel to use for which message.

---

## What You CAN Use AI For (75%)

- GitHub Actions YAML (great at this).
- Docker publishing steps.
- Reused code for projects 5 and 6.
- Tests and fixtures.

Forbidden:
- Deciding which branches trigger deploys.
- Storing secrets in workflow files (always GitHub Secrets).
- Writing state machines without paper first.

---

## Things AI Is Bad At This Week

- **Secrets in workflows.** AI will sometimes echo env vars in logs. Always use masking.
- **Branch protection rules.** These are repo settings, not workflow files. AI cannot help you configure them.
- **Environment-specific deploys.** Staging vs prod distinction is often missed. Be explicit.

---

## Core Mental Models For This Week

- **A workflow is a sequence of jobs, each a sequence of steps.**
- **A secret is a value GitHub holds for you and never logs.** Everything secret should be a GitHub Secret.
- **CI is a promise: "if main is green, main is deployable".** Protect that promise with branch rules.

---

## This Week's AI Audit Focus

For every workflow file, list: what triggers it, what secrets it uses, which environments it touches, and who reviewed the deploy destinations.

---

## Assessment

- Push a change. Watch CI run. Verify the deploy lands in staging.
- Facilitator asks: "what prevents a force-push to main from bypassing your CI?" You answer with the specific branch protection rule.

---

## One Sentence To Remember

"A pipeline that deploys the wrong thing silently is worse than no pipeline at all."
