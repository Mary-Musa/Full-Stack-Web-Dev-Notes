# Week 19 - Day 4 Assignment

## Title
Publish The Package To npm (Or A Private Registry)

## Overview
Day 4 is pre-weekend ship day. Today you publish your payments package. If you are nervous about real npm, publish to a private registry (GitHub Packages) or use `npm pack` to create a tarball you can install locally. Either way, the goal is the same: your package is installable by `npm install`.

## Learning Objectives Assessed
- Publish a package to an npm registry
- Use scoped package names (`@mctaba/payments`)
- Install the published package from a separate project
- Understand the publish workflow

## Prerequisites
- Days 1-3 completed

## AI Usage Rules

**Ratio:** 35/65. **Habit:** You design the API. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Publishing command walkthroughs.
- **NOT ALLOWED FOR:** Running publish commands without understanding them.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Pick a registry

**What to do:**
Pick one:
- **npm public** -- `npmjs.com`. Free. Public. Real.
- **GitHub Packages** -- free for public repos, paid for private.
- **Verdaccio** -- self-hosted, good for learning.
- **npm pack** -- creates a local tarball, no registry at all.

For first-timers I recommend `npm pack` for the learning and maybe GitHub Packages for the real publish.

**Expected output:**
Decision recorded in `publish-notes.md` with reasoning.

### Task 2: Log in and publish

**What to do:**
For public npm:

```bash
cd packages/mctaba-payments
npm login
npm publish --access public
```

For GitHub Packages, add to `package.json`:

```json
"publishConfig": { "registry": "https://npm.pkg.github.com" }
```

And authenticate with a GitHub personal access token.

For `npm pack`:

```bash
npm pack
# creates mctaba-payments-0.1.0.tgz
```

**Expected output:**
Package published (or tarball created). Screenshot `day4-publish.png`.

### Task 3: Install from a fresh project

**What to do:**
In a new folder (not your monorepo):

```bash
mkdir /tmp/test-consumer && cd /tmp/test-consumer
npm init -y
npm install @mctaba/payments
```

Or if using the tarball:

```bash
npm install ~/path/to/mctaba-payments-0.1.0.tgz
```

Create `test.js`:

```javascript
import { createClient } from "@mctaba/payments";
const client = createClient({});
console.log(typeof client.initiatePayment); // "function"
```

Run it with Node.

**Expected output:**
Import works. Console logs "function".

### Task 4: Bump version and republish

**What to do:**
Make a small change in the package (comment, docs, whatever). Run:

```bash
npm version patch    # 0.1.0 -> 0.1.1
npm publish
```

Install the new version in your test consumer. Verify you get 0.1.1.

**Expected output:**
Two versions on the registry (or two tarballs).

### Task 5: Publish workflow notes

**What to do:**
In `publish-notes.md`, write 5-7 sentences:
- What is the difference between `npm pack` and `npm publish`?
- Why are scoped packages (`@mctaba/...`) different from regular ones?
- What is the difference between a public and private package?
- What happens if you publish something you did not mean to? (Hint: `npm unpublish` within 72 hours.)

Your own words.

**Expected output:**
`publish-notes.md` complete.

## Stretch Goals (Optional - Extra Credit)

- Set up a GitHub Actions workflow that publishes on tag push.
- Use `semantic-release` to fully automate publishing.
- Add a `README.md` screenshot that appears on the npm package page.

## Submission Requirements

- **What to submit:** Repo, screenshots, `publish-notes.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 4

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Package published to chosen registry | 30 | Visible in registry OR tarball produced. |
| Install in a fresh consumer works | 25 | Screenshot confirms. |
| Version bump and republish | 20 | Two versions exist. |
| Publish notes | 20 | Four questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Publishing with `private: true`.** The package will refuse to publish. That flag is for safety in monorepos.
- **Forgetting to bump the version.** `npm publish` fails if the version is already published.
- **Publishing secrets.** The `files` array in package.json defaults to everything. Be specific.

## Resources

- Day 4 reading: [Publishing the Package.md](./Publishing%20the%20Package.md)
- Week 19 AI boundaries: [../ai.md](../ai.md)
- npm publish docs: https://docs.npmjs.com/cli/v9/commands/npm-publish
