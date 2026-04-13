# Week 11 - Day 5 Assignment

## Title
Monorepo Structures And Workspace Dependencies

## Overview
Week 11 Day 5 is a Git-adjacent infrastructure day. Your CRM has grown two distinct parts (backend + frontend) that belong in one project. Today you restructure the repository as a monorepo using npm workspaces, learn when this is a good idea (and when it is not), and ship a clean push that other cohort members can clone and run with one command.

## Learning Objectives Assessed
- Understand what a monorepo is and why it exists
- Configure npm workspaces in a root `package.json`
- Run scripts across multiple packages from the root
- Write a root README that documents the full project

## Prerequisites
- Weeks 1-10 Day 5 completed
- Week 11 Days 1-4 completed (backend + frontend both working)

## AI Usage Rules

**Ratio:** 45/55. **Habit:** New tech = manual handshake. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Generating the root README after your structure is in place.
- **NOT ALLOWED FOR:** Configuring `package.json` workspaces without understanding each field.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Reorganise the repo

**What to do:**
Move your existing code into a monorepo structure:

```
whatsapp-crm/
  package.json          (root, workspaces config)
  packages/
    backend/            (your Express server)
      package.json
      index.js
      ...
    frontend/           (your React dashboard)
      package.json
      src/
      ...
  README.md
```

Use `git mv` to preserve history:

```bash
mkdir -p packages/backend packages/frontend
git mv routes services db index.js packages/backend/
git mv ... packages/frontend/
```

**Expected output:**
Repo restructured. Commit with message `refactor: move to monorepo layout`.

### Task 2: Configure npm workspaces

**What to do:**
In the root `package.json`:

```json
{
  "name": "whatsapp-crm",
  "private": true,
  "workspaces": [
    "packages/*"
  ],
  "scripts": {
    "dev:backend": "npm run dev -w @crm/backend",
    "dev:frontend": "npm run dev -w @crm/frontend",
    "dev": "concurrently \"npm:dev:backend\" \"npm:dev:frontend\""
  },
  "devDependencies": {
    "concurrently": "^8.0.0"
  }
}
```

In `packages/backend/package.json`, set `"name": "@crm/backend"`. Same for frontend.

Run `npm install` at the root. Verify both packages install their deps.

**Expected output:**
One `npm install` at the root installs everything. `npm run dev` starts both servers.

### Task 3: Root README

**What to do:**
Write `README.md` at the root documenting:
- What the project is (2 sentences)
- Prerequisites (Node, ngrok, Meta dev account)
- Setup (env vars, install, first run)
- Architecture diagram (text or image)
- Common commands (dev, test, build)
- Troubleshooting (3-5 common issues)

A stranger cloning the repo should be able to run it following this README.

**Expected output:**
Root `README.md` committed.

### Task 4: Shared TypeScript / lint config (optional in JS)

**What to do:**
Add a shared `.eslintrc.js` at the root that both packages inherit. Or a shared `.prettierrc`. This is a simple way to keep code style consistent across the monorepo.

For a JS-only project, a single `.prettierrc` at the root is enough.

**Expected output:**
Shared config visible. Both packages format the same way.

### Task 5: Monorepo reflection

**What to do:**
In `monorepo-notes.md`, write 5-8 sentences answering:
- Why is a monorepo better than two separate repos for this project?
- When would two separate repos be better? (Hint: different deployment cadences, different teams.)
- What is the cost of a monorepo? (Hint: CI complexity, lockfile churn, unfamiliarity.)

Your own words.

**Expected output:**
`monorepo-notes.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Try pnpm workspaces or Turborepo for faster builds.
- Add a `packages/shared/` package with types or helpers used by both backend and frontend.
- Set up a CI workflow that tests both packages on every push.

## Submission Requirements

- **What to submit:** Repo with monorepo layout, root README, `monorepo-notes.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 5

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Repo restructured cleanly | 20 | Two packages under `packages/`. History preserved via `git mv`. |
| npm workspaces configured | 25 | One `npm install` at root works. Scripts run both services. |
| Root README | 20 | Complete setup instructions. A stranger could clone and run. |
| Shared config | 10 | Prettier or ESLint config at root inherited by both. |
| Monorepo reflection notes | 15 | Three questions answered honestly. |
| Clean commits | 10 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Moving files with `mv` instead of `git mv`.** `mv` + `git add` loses rename history. `git mv` preserves it.
- **Forgetting `"private": true` on the root package.** Without it, npm thinks you are trying to publish the monorepo.
- **Duplicating dependencies in every package.** Shared deps go in the root `devDependencies` where possible.

## Resources

- Prior Day 5 assignments (Weeks 1-10)
- Week 11 AI boundaries: [../ai.md](../ai.md)
- npm workspaces: https://docs.npmjs.com/cli/v9/using-npm/workspaces
