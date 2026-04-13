# Week 14 - Day 5 Assignment

## Title
Deploying Next.js To Vercel (Or Alternatives) And A Preview Environment Workflow

## Overview
Week 14 Day 5 ships your Next.js shop to a real URL. Vercel is the natural home for Next.js; Railway, Render, and Netlify all work too. You will deploy the main branch, connect a real Postgres (or use a hosted one for free), and learn the preview deploy workflow that real teams use on every PR.

## Learning Objectives Assessed
- Deploy a Next.js app to a free cloud provider
- Configure environment variables in the cloud
- Understand preview deploys per PR
- Use a hosted Postgres (Neon, Supabase, or Railway) for the shop

## Prerequisites
- Weeks 1-13 Day 5 completed

## AI Usage Rules

**Ratio:** 40/60. **Habit:** Framework docs first. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Vercel dashboard guidance and env var examples.
- **NOT ALLOWED FOR:** Pasting database credentials into your repo.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Hosted Postgres

**What to do:**
Pick one:
- **Neon** (https://neon.tech) -- free tier, excellent for Postgres
- **Supabase** (https://supabase.com) -- free tier, Postgres + more
- **Railway** (https://railway.app) -- pay-as-you-go, dead simple

Create a database. Copy the connection string.

Load your `schema.sql` into it. Seed with a few products.

**Expected output:**
Hosted database reachable from your laptop.

### Task 2: Deploy to Vercel

**What to do:**
1. Push your repo to GitHub.
2. Sign up at https://vercel.com and connect your GitHub.
3. Import the repo. Select the `packages/shop-next` folder as the root (if using a monorepo).
4. Set environment variables: `PG_HOST`, `PG_PORT`, `PG_USER`, `PG_PASSWORD`, `PG_DATABASE` from your hosted Postgres.
5. Click Deploy. Wait 2-3 minutes.

**Expected output:**
Live URL. Screenshot `day5-live.png`.

### Task 3: Preview deploys

**What to do:**
1. Create a feature branch `feature/tagline`.
2. Make a small change (update a heading).
3. Push the branch and open a PR on GitHub.
4. Vercel automatically creates a preview deploy for the PR. A comment appears on the PR with the preview URL.
5. Click the preview URL. Your change is live at a unique URL.

**Expected output:**
Screenshot `day5-preview.png` of the PR with the Vercel comment.

### Task 4: Merge and auto-deploy

**What to do:**
Merge the PR to main. Watch Vercel auto-deploy main. The production URL now shows your change.

**Expected output:**
Production URL updated.

### Task 5: Deployment notes

**What to do:**
In `deployment-notes.md`, answer:
- What is the advantage of per-PR preview deploys?
- What is the risk of storing DB credentials in Vercel env vars?
- How would you rotate a leaked credential without downtime?
- What happens if your hosted DB is down?

Your own words.

**Expected output:**
`deployment-notes.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Add a custom domain to Vercel (free with some registrars).
- Enable Vercel Analytics (free tier).
- Set up a `staging` branch with its own preview alias.

## Submission Requirements

- **What to submit:** Live URL, PR link, screenshots, `deployment-notes.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 5

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Hosted Postgres working | 15 | App connects in production. |
| Production deploy live | 30 | URL accessible, products render. |
| Preview deploy on PR | 25 | Screenshot confirms. |
| Merge triggers production update | 10 | Verified. |
| Deployment notes | 15 | Four questions answered honestly. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Committing `.env.local`.** Vercel reads env vars from its dashboard, not from a committed file. Never commit real credentials.
- **Using a free tier for real traffic.** Free tiers throttle. For real customers, pay for a paid tier.
- **Assuming the build works after "it worked on my machine".** Always deploy early and iterate.

## Resources

- Prior Day 5 assignments
- Week 14 AI boundaries: [../ai.md](../ai.md)
- Vercel Next.js deploy: https://vercel.com/docs/frameworks/nextjs
- Neon: https://neon.tech/
