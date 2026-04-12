# Week 19, Day 4: Publishing the Package

By the end of today, `mctaba-payments` is a real installable package from a real registry -- either npmjs.org or GitHub Packages. Your shop installs it via `npm install mctaba-payments`, no file paths, no copy-paste. You understand versioning, changelogs, and the subtle ways a published package can bite you.

**Prior concepts:** the package from Days 1-3.

**Estimated time:** 2-3 hours

---

## Where To Publish

You have three options:

1. **npmjs.org (public)** -- anyone on earth can install. Free. Simple. Eternal -- you cannot delete a version after 72 hours, because the entire npm ecosystem depends on published versions not disappearing.
2. **GitHub Packages (private)** -- only people with access to your GitHub org can install. Free for small orgs. Good for internal libraries.
3. **Git install** -- `npm install git+https://github.com/you/mctaba-payments.git`. No registry involved. Works for small teams; falls apart at scale.

For the Marathon we will publish publicly on npmjs.org. This:
- Teaches you the real workflow.
- Adds a portfolio item.
- Costs nothing.
- Forces you to care about quality because strangers can read your code.

If your payments code has secrets or company-sensitive logic, pick GitHub Packages instead.

---

## Pre-Publish Checklist

Before `npm publish`:

- [ ] `package.json` has a unique name (`mctaba-payments` may be taken; pick `mctaba-payments-YOURNAME` or your own scoped name).
- [ ] `package.json` has correct version, main, files, keywords, repository, author.
- [ ] `README.md` is complete.
- [ ] `LICENSE` file exists. MIT is the default for libraries.
- [ ] `.npmignore` or `"files"` in package.json lists only what should ship -- no tests, no `.env`, no `node_modules`.
- [ ] All tests pass.
- [ ] No `console.log` calls in src.
- [ ] No TODO comments in src.
- [ ] Version starts at `0.1.0` (semver pre-1.0 means "breaking changes allowed at any version bump").

### The `files` field

In `package.json`:

```json
{
  "name": "@mctaba/payments",
  "version": "0.1.0",
  "main": "index.js",
  "files": [
    "index.js",
    "src/",
    "sql/",
    "README.md",
    "SECURITY.md",
    "LICENSE"
  ],
  "keywords": ["payments", "m-pesa", "airtel", "stripe", "kenya"],
  "repository": {
    "type": "git",
    "url": "https://github.com/you/mctaba-payments.git"
  },
  "author": "Your Name",
  "license": "MIT"
}
```

The `files` whitelist is stricter than `.npmignore` blacklist and is the modern recommendation. Nothing ships unless it is listed.

### Scoped packages

A name like `@mctaba/payments` is a **scoped package**. It lives under your org (or username) scope. Benefits:

1. No name collisions -- `@mctaba/payments` is always yours regardless of other `payments` packages.
2. You can publish private scoped packages on npmjs.org if you pay for npm Teams.
3. The scope acts as branding.

Scoped packages publish with `npm publish --access public` (without the flag, they default to private which requires a paid plan).

---

## First Publish

```bash
# Create an npmjs.org account if you do not have one
npm adduser
# Enter username, password, email, one-time code

# Verify
npm whoami

# Publish
npm publish --access public
```

Within seconds, `https://www.npmjs.com/package/@yourscope/payments` is live. Anyone can `npm install @yourscope/payments`.

### Sanity check the published version

Create a throwaway folder, install your package, and import it. If it works, you are done.

```bash
cd /tmp
mkdir test-install
cd test-install
npm init -y
npm install @yourscope/payments
node -e "console.log(require('@yourscope/payments'))"
```

You should see the module's exported API printed.

---

## Replacing The Local Install In The Shop

Back in your shop's `server/package.json`, remove the `file:../mctaba-payments` entry:

```bash
cd server
npm uninstall mctaba-payments
npm install @yourscope/payments
```

Update the import:

```javascript
// server/index.js
const payments = require("@yourscope/payments").init({ pool, config });
```

Run the shop. Everything should still work. The code no longer references the local folder -- the dependency is real.

---

## Versioning And Semver

Semantic versioning (semver) is the contract every library author makes with their users:

- `MAJOR.MINOR.PATCH`
- Bump PATCH for bug fixes that do not change the API (`0.1.0` -> `0.1.1`).
- Bump MINOR for new features that do not break existing users (`0.1.1` -> `0.2.0`).
- Bump MAJOR for breaking changes (`0.2.0` -> `1.0.0`).

Pre-1.0 (`0.x.y`) is a special zone: breaking changes are allowed at any bump. Once you go to `1.0.0` you are committing to not breaking users without a major bump.

Publishing a new version:

```bash
# Make a change, update CHANGELOG.md
npm version patch   # bumps to 0.1.1, commits, tags
npm publish --access public
```

`npm version` auto-bumps, commits, and tags. It is the only right way -- do not edit `package.json` by hand.

---

## Changelog

Start a `CHANGELOG.md` now and maintain it religiously:

```markdown
# Changelog

## 0.2.0 (2026-04-30)

### Added
- Refund support for Airtel Money.
- Reconciliation admin API.

### Changed
- `initiate` now validates phone format before calling the provider.

### Fixed
- Duplicate Stripe webhooks no longer fire double WhatsApp notifications.

## 0.1.0 (2026-04-15)

Initial release. Supports M-Pesa, Airtel, and Stripe.
```

Users read this before upgrading. It tells them what to expect.

---

## Deprecations

When you need to remove a function, do not remove it. **Deprecate** it first:

```javascript
function oldFunctionName(args) {
  console.warn("mctaba-payments: oldFunctionName is deprecated. Use newFunctionName instead. Will be removed in 1.0.0.");
  return newFunctionName(args);
}
```

Give users at least one minor version between deprecation and removal. A ten-line library does not need this discipline; a payments library ten shops depend on does.

---

## Security Releases

If you find a security issue in an already-published version, you must:

1. Do not announce the issue publicly until a fix is available.
2. Publish a patch version with the fix.
3. Write a GitHub Security Advisory with the CVE (if severe).
4. Deprecate all vulnerable versions:

```bash
npm deprecate @yourscope/payments@<0.2.3 "Upgrade to 0.2.3: fixes CVE-YYYY-NNNNN signature bypass"
```

Users installing old vulnerable versions see the deprecation warning.

---

## Checkpoint

1. `npm publish --access public` succeeds.
2. The package page on npmjs.org renders the README.
3. A fresh project can `npm install @yourscope/payments` and use the exported API.
4. The shop installs it from the registry, not from `file:`.
5. `npm version patch && npm publish` bumps and republishes cleanly.
6. `CHANGELOG.md` exists with at least one entry.
7. You have a plan for deprecations and security releases.

Commit:

```bash
git add .
git commit -m "chore: publish 0.1.0 to npm and add changelog"
git push --follow-tags
```

---

## What You Learned

- Public npm publishing is fast and free.
- Scoped packages avoid name collisions.
- `npm version` is the only right way to bump.
- Changelogs are required; users read them.
- Semver 0.x allows breaking changes; 1.0+ does not.
- Security releases have their own discipline.

Tomorrow: recap and peer review. The weekend builds a small demo app that uses your package from scratch.
