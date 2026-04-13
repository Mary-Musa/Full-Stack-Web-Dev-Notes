# Week 25 - Day 2 Assignment

## Title
Docker Compose -- Multi-Container Dev Environment

## Overview
Today you take the compose file from Week 24 and treat it like a first-class citizen: named services, depends_on with health, shared networks, and `compose up -d` for daily use.

## Learning Objectives Assessed
- Manage multi-container environments with compose
- Use `depends_on: condition: service_healthy`
- Expose the right ports only
- Use compose profiles for optional services

## Prerequisites
- Day 1 completed
- Week 24 compose file

## AI Usage Rules

**Ratio this week:** 30% manual / 70% AI
**Habit:** AI writes infra, you verify boundaries. See [../ai.md](../ai.md).

- **ALLOWED FOR:** YAML boilerplate.
- **NOT ALLOWED FOR:** Deciding which ports are exposed to the host.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Healthchecks

**What to do:**
Add healthchecks to postgres and redis:

```yaml
postgres:
  image: postgres:16
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 5s
    timeout: 3s
    retries: 5
redis:
  image: redis:7
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 5s
    timeout: 3s
    retries: 5
```

And on the api:

```yaml
api:
  depends_on:
    postgres: { condition: service_healthy }
    redis: { condition: service_healthy }
```

**Expected output:**
API does not start until DB and Redis are ready.

### Task 2: Only expose what you need

**What to do:**
Remove `ports:` from postgres and redis (they should only be reachable on the internal compose network). Keep `ports: ["3000:3000"]` on api.

**Expected output:**
`docker compose ps` shows only api exposes a port to the host.

### Task 3: Profiles

**What to do:**
Add an optional `bullboard` service behind a profile:

```yaml
bullboard:
  image: deadly0/bull-board
  profiles: ["dev"]
  ports: ["3030:3000"]
```

Start it with `docker compose --profile dev up`. Without the flag, it stays off.

**Expected output:**
Profile works.

### Task 4: Network diagram

**What to do:**
In `network.md`, sketch or describe:
- Which services talk to which
- Which are reachable from the host
- Which are internal only

Draw with ASCII or upload a picture.

**Expected output:**
Committed.

### Task 5: Local ergonomics

**What to do:**
Add `docker compose logs -f api` to your `package.json` as `"logs": "docker compose logs -f api"` and a few sibling scripts (`db:shell`, `redis:shell`, `down`).

**Expected output:**
Scripts committed and documented in README.

## Stretch Goals (Optional - Extra Credit)

- Use `docker compose watch` for hot reload.
- Add a separate `test` profile that seeds a test database.
- Use `x-anchor` to share env vars across services.

## Submission Requirements

- **What to submit:** Repo with compose file, network notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 2

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Healthchecks | 25 | api waits for services. |
| Minimal exposed ports | 20 | Only api on 3000. |
| Profiles | 15 | Optional service works. |
| Network notes | 20 | Clear diagram. |
| Local scripts | 15 | Scripts in package.json. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Exposing postgres on 5432 in dev.** Fine for local-only but causes conflict when another Postgres is installed.
- **No healthchecks.** API starts too early, crashes because DB is not ready.
- **Profiles for required services.** Only optional ones.

## Resources

- Day 2 reading: [Docker Compose.md](./Docker%20Compose.md)
- Week 25 AI boundaries: [../ai.md](../ai.md)
