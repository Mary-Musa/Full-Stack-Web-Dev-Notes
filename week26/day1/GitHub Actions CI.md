# Week 26, Day 1: GitHub Actions for CI

> **AI boundaries this week:** 25% manual / 75% AI. Habit: *AI scaffolds, you review the pipeline.* See [ai.md](../ai.md).

By the end of today, every push to your repo runs tests, lints, and builds Docker images automatically. Failed checks block merges. You will have green checkmarks on your PRs and know when code breaks before it reaches production.

**Prior-week concepts you will use today:**
- The test suites from Week 19 (payments package).
- The Docker builds from Week 25.
- The two-service split from Week 24.

**Estimated time:** 3 hours

---

## The Week Ahead

| Day | Focus |
|---|---|
| Day 1 (today) | GitHub Actions basics: test and lint on push. |
| Day 2 | Building and publishing Docker images to a registry. |
| Day 3 | Deploying automatically to a server. |
| Day 4 | Project 5: Booking & Appointments System. |
| Day 5 | Project 6 assembly and phase 4 recap. |
| Weekend | Ship projects 5 and 6. |

---

## What CI Actually Is

Continuous Integration: every code change goes through automated verification before merging. The workflow is:

1. Developer pushes a branch.
2. GitHub Actions runs your test suite, linter, and build.
3. If everything passes, a green check appears on the PR.
4. Reviewer merges.
5. If anything fails, the PR is blocked until fixed.

The goal: catch broken code at PR time, not after it is merged and breaking production.

---

## Your First Workflow

GitHub Actions uses YAML files in `.github/workflows/`. Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: |
            api/package-lock.json
            notification-service/package-lock.json
            shop/package-lock.json

      - name: Install API dependencies
        working-directory: ./api
        run: npm ci

      - name: Install notification-service dependencies
        working-directory: ./notification-service
        run: npm ci

      - name: Install shop dependencies
        working-directory: ./shop
        run: npm ci

      - name: Run API tests
        working-directory: ./api
        env:
          DB_HOST: localhost
          DB_PORT: 5432
          DB_USER: test_user
          DB_PASSWORD: test_password
          DB_NAME: test_db
          REDIS_URL: redis://localhost:6379
        run: npm test

      - name: Run notification-service tests
        working-directory: ./notification-service
        run: npm test

      - name: Build shop
        working-directory: ./shop
        run: npm run build
```

Long file, but each section is obvious once you read it:

- **Triggers**: pushes and PRs to main.
- **Runs on**: a fresh Ubuntu VM spun up by GitHub.
- **Services**: Postgres and Redis are available as if they were on localhost, same as your dev environment.
- **Steps**: check out the code, set up Node, install deps for each service, run tests, build.

The whole thing takes 3-5 minutes on a cold cache. GitHub caches `node_modules` between runs if you point at lockfiles.

---

## Your First Failing Build

Push a commit that breaks a test on purpose. Go to the "Actions" tab on GitHub. Watch the workflow run. Click it. See exactly which step failed and the error message.

This is the whole value: a failing test is visible to everyone, impossible to ignore, and blocks merges until someone fixes it. You cannot "just push" broken code to main.

---

## Linting Before Tests

Add a lint step early in the workflow -- linting fails fast and saves minutes on runs with style errors:

```yaml
- name: Lint API
  working-directory: ./api
  run: npm run lint || true  # soft fail for now

- name: Lint shop
  working-directory: ./shop
  run: npm run lint
```

If you have not set up eslint, do it now:

```bash
cd api
npm install --save-dev eslint
npx eslint --init
```

Pick a standard config (Standard, Airbnb, or the default). Add `"lint": "eslint ."` to `package.json`.

For the shop (Next.js), `npm run lint` is already wired by `create-next-app`.

---

## Branch Protection

On GitHub, go to Settings -> Branches -> Add rule for `main`:

- Require a pull request before merging.
- Require status checks to pass before merging (pick "test" from your workflow).
- Require branches to be up to date.
- Include administrators (optional -- prevents you from bypassing).

Now a PR with failing CI cannot be merged. This is the enforcement layer. Without it, CI is advisory. With it, CI is law.

---

## Matrix Builds (Optional)

If you want to test against multiple Node versions:

```yaml
strategy:
  matrix:
    node-version: [18, 20, 21]

steps:
  - uses: actions/setup-node@v4
    with:
      node-version: ${{ matrix.node-version }}
```

Runs the whole workflow three times, once per Node version. Useful for libraries (like `@mctaba/payments`) where users might be on different Node versions. Not necessary for apps that pin a single version.

---

## Secrets In Workflows

Some tests need real secrets (e.g., Stripe test key). Never put them in the YAML.

Go to Settings -> Secrets and variables -> Actions -> New repository secret. Add `STRIPE_TEST_SECRET_KEY`.

In the workflow:

```yaml
- name: Run Stripe integration tests
  env:
    STRIPE_TEST_SECRET_KEY: ${{ secrets.STRIPE_TEST_SECRET_KEY }}
  run: npm test -- --grep "stripe"
```

Secrets are masked in logs. Even if your test accidentally `console.log`s the secret, GitHub Actions replaces it with `***`.

---

## Caching

The first run of a workflow is slow because npm downloads everything. Subsequent runs can reuse cached `node_modules`:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 20
    cache: npm
    cache-dependency-path: api/package-lock.json
```

The `cache: npm` option with the `cache-dependency-path` tells Actions to cache based on the lockfile. If the lockfile has not changed, the cache hits and `npm ci` is much faster.

For Docker image builds, use `docker/build-push-action` with `cache-from: type=gha` (GitHub Actions cache).

---

## What To Run, What Not To

Your CI should include:

- **Unit tests**: fast, deterministic, no external dependencies (mocks are fine).
- **Integration tests against Postgres + Redis**: reasonable, tested above.
- **Linting**: cheap, catches style regressions.
- **Type checking** (if TypeScript): cheap, catches type regressions.
- **Build**: ensures code compiles. Already tested above.

CI should **not** include:

- **Payment provider integration tests**: too fragile, sandboxes go down, rate limits, secret handling complications. Run these manually before releases.
- **Smoke tests against production**: those are post-deploy, not CI.
- **Visual regression tests**: expensive, flaky. Different tool entirely.

Fast, deterministic, focused on "does this PR break what we know about". That is the CI contract.

---

## Checkpoint

1. `.github/workflows/ci.yml` is in place.
2. Pushing a branch triggers the workflow on GitHub.
3. The workflow runs tests against real Postgres and Redis.
4. A test failure blocks the PR from merging.
5. Lint errors appear as PR annotations with file and line.
6. Cached node_modules make the second run faster.
7. Branch protection prevents direct pushes to main.

Commit:

```bash
git add .
git commit -m "ci: github actions for tests lint and build"
git push
```

---

## What You Learned

- GitHub Actions workflows are YAML in `.github/workflows/`.
- Services (Postgres, Redis) run alongside your tests in the same VM.
- Branch protection turns CI advice into enforcement.
- Secrets live in GitHub, masked in logs.
- Caching makes repeat runs fast.
- CI should be fast and deterministic; integration tests against real providers do not belong here.

Tomorrow we build and publish Docker images automatically.
