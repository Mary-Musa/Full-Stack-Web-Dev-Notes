# Week 25 - Day 3 Assignment

## Title
Volumes, Environments, And Networks

## Overview
Day 3 is the persistence layer. You learn volumes (so Postgres data survives container restarts), environment files for different stages, and custom networks for isolation.

## Learning Objectives Assessed
- Use a named volume for Postgres data
- Separate dev vs test configuration
- Create custom networks and attach services
- Back up and restore a volume

## Prerequisites
- Days 1-2 completed

## AI Usage Rules

**Ratio this week:** 30% manual / 70% AI
**Habit:** AI writes infra, you verify boundaries. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Volume and network boilerplate.
- **NOT ALLOWED FOR:** Deciding your backup strategy.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Named Postgres volume

**What to do:**
```yaml
services:
  postgres:
    volumes:
      - pg-data:/var/lib/postgresql/data
volumes:
  pg-data:
```

Restart compose. Confirm data survives: insert a row, stop, start, row still there.

**Expected output:**
Data persistent across restarts.

### Task 2: Env files per stage

**What to do:**
Split env into `.env.dev` and `.env.test`. Start with:

```bash
docker compose --env-file .env.dev up
docker compose --env-file .env.test up
```

Commit `.env.dev.example` and `.env.test.example` with placeholders.

**Expected output:**
Both environments start cleanly.

### Task 3: Custom network

**What to do:**
Create two networks:

```yaml
networks:
  internal:
  public:
```

Attach postgres and redis to `internal` only. Attach api to both. The notifications service only to `internal`.

**Expected output:**
`docker network inspect` shows the isolation.

### Task 4: Volume backup

**What to do:**
Run a one-shot container that dumps the DB:

```bash
docker compose exec postgres pg_dump -U postgres mctaba > backup.sql
```

Restore it to a fresh container:

```bash
cat backup.sql | docker compose exec -T postgres psql -U postgres mctaba
```

**Expected output:**
`backup.sql` committed (small dummy data). Restore works.

### Task 5: Persistence notes

**What to do:**
In `persistence-notes.md`, write 5-7 sentences:
- What is the difference between a named volume and a bind mount?
- Why use a named volume for Postgres?
- What happens to an anonymous volume if you `docker compose down -v`?
- How often should you back up in production? (Frequently. At least daily.)

Your own words.

**Expected output:**
`persistence-notes.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Write a `scripts/backup.sh` that timestamps the dump.
- Mount a bind-mount for the `uploads/` folder.
- Try Docker Desktop's volume UI.

## Submission Requirements

- **What to submit:** Repo with volumes, backup, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 3

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Named volume | 20 | Data persists. |
| Env files per stage | 15 | Two env files. |
| Custom network isolation | 25 | Inspected. |
| Volume backup + restore | 25 | Tested. |
| Persistence notes | 10 | Four questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **`docker compose down -v`.** This deletes volumes. Never run in prod.
- **Bind mount for databases.** Permissions/perf issues on Mac/Windows.
- **No backup policy.** A database without backups is a tragedy waiting to happen.

## Resources

- Day 3 reading: [Volumes Environments and Networks.md](./Volumes%20Environments%20and%20Networks.md)
- Week 25 AI boundaries: [../ai.md](../ai.md)
