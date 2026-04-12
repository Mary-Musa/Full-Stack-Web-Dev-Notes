# Week 25, Day 3: Volumes, Environments, and Networks

By the end of today, you have a clean pattern for dev vs production compose files, understand how to isolate services on different networks, and know how to move volume data between machines.

**Prior concepts:** compose basics (Week 25 Day 2).

**Estimated time:** 2 hours

---

## Dev vs Production Compose Files

You want two slightly different setups:

- **Dev**: bind mounts for hot-reloading, ports exposed for localhost access, debug logging, `NODE_ENV=development`.
- **Production**: no bind mounts, internal-only ports (only Nginx exposed), minimal logging, `NODE_ENV=production`.

Compose supports this with **override files**:

`docker-compose.yml` -- the base, production-ready.
`docker-compose.override.yml` -- dev-only overrides, automatically merged when you run `docker-compose up` without flags.
`docker-compose.prod.yml` -- production-only overrides, applied with `docker-compose -f docker-compose.yml -f docker-compose.prod.yml up`.

Example split:

**`docker-compose.yml`** (base -- production-ready):

```yaml
services:
  api:
    build: ./api
    restart: unless-stopped
    depends_on: [postgres, redis]
    environment:
      NODE_ENV: production
```

**`docker-compose.override.yml`** (dev):

```yaml
services:
  api:
    ports:
      - "5000:5000"
    volumes:
      - ./api:/app
      - /app/node_modules
    command: npm run dev
    environment:
      NODE_ENV: development
      DB_LOG: "true"

  postgres:
    ports:
      - "5432:5432"

  redis:
    ports:
      - "6379:6379"
```

Now:

- `docker-compose up` in dev mode: auto-merges override, dev-friendly.
- `docker-compose -f docker-compose.yml up` in production mode: base only.

Keep secrets out of both files. They live in a host-level `.env` referenced with `${VAR}` substitution.

---

## Named Networks

By default, all services in one compose file share one network. You can split them for better isolation:

```yaml
services:
  postgres:
    networks:
      - backend

  redis:
    networks:
      - backend

  api:
    networks:
      - backend
      - frontend

  notifications:
    networks:
      - backend

  shop:
    networks:
      - frontend

  nginx:
    networks:
      - frontend
    ports:
      - "80:80"
      - "443:443"

networks:
  backend:
  frontend:
```

- **`backend`** network has Postgres, Redis, API, notifications. No external access. Only services in this network can reach the database.
- **`frontend`** network has the API, shop, and Nginx. This is what Nginx routes to.
- The API is in both networks -- it bridges frontend to backend.
- Nothing else can reach Postgres or Redis from outside.

This matters for production. If a container gets compromised, an attacker cannot reach the database unless they are in the `backend` network. Defense in depth.

---

## Volume Backups

Your Postgres data lives in a named volume. Backing it up:

```bash
# Create a backup
docker run --rm \
  -v your-repo_postgres_data:/source \
  -v $(pwd):/backup \
  alpine tar czf /backup/postgres-backup-$(date +%Y%m%d).tar.gz -C /source .

# Restore from backup
docker run --rm \
  -v your-repo_postgres_data:/target \
  -v $(pwd):/backup \
  alpine tar xzf /backup/postgres-backup-20260415.tar.gz -C /target
```

Back up before every major migration. Restore on a fresh machine to verify the backup works.

Better: schedule `pg_dump` via cron and ship the SQL file to S3 or similar. A full tar backup works but is larger and harder to partially restore.

---

## Resource Limits

Stop any container from eating the whole machine:

```yaml
api:
  deploy:
    resources:
      limits:
        cpus: "1.0"
        memory: 512M
      reservations:
        memory: 128M
```

One CPU core, 512 MB memory cap, 128 MB reserved. If the API leaks memory, it gets OOM-killed instead of taking down the host.

Tune these based on load tests. For a Kenyan SME shop serving a few hundred customers, 0.5 CPU and 256 MB is usually enough.

---

## Logs Rotation

By default Docker keeps all container logs forever. Disk fills up. Configure log rotation:

```yaml
api:
  logging:
    driver: "json-file"
    options:
      max-size: "10m"
      max-file: "5"
```

10 MB per log file, keep 5 files, rotate. 50 MB total per container. Apply to every service.

For production, ship logs to a central system (ELK, Loki, CloudWatch). For the Marathon, rotation is enough.

---

## Healthchecks Across All Services

Every service should have a healthcheck. Compose will not start a dependent service until its dependencies are healthy if you use `condition: service_healthy`. This prevents races at startup.

A universal pattern for Node services:

```yaml
api:
  healthcheck:
    test: ["CMD", "wget", "-q", "-O-", "http://localhost:5000/health"]
    interval: 10s
    timeout: 3s
    retries: 3
    start_period: 15s
```

For Postgres:

```yaml
postgres:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U crm_user -d crm"]
    interval: 5s
    timeout: 3s
    retries: 5
```

For Redis:

```yaml
redis:
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 5s
    timeout: 3s
    retries: 5
```

Every service needs one. This is non-negotiable for production.

---

## Secrets

Compose has a `secrets` feature:

```yaml
services:
  api:
    secrets:
      - jwt_secret
      - mpesa_passkey

secrets:
  jwt_secret:
    file: ./secrets/jwt_secret.txt
  mpesa_passkey:
    file: ./secrets/mpesa_passkey.txt
```

Secrets get mounted into the container at `/run/secrets/<name>` as files. Your code reads them at startup.

For the Marathon we use env vars and a gitignored `.env`. Proper secrets management comes with real production -- HashiCorp Vault, AWS Secrets Manager, etc. Compose's built-in secrets are a middle ground.

---

## Checkpoint

1. `docker-compose.yml` + `docker-compose.override.yml` gives you a dev setup with hot reload.
2. `docker-compose -f docker-compose.yml up` runs the production-shaped setup (no bind mounts, no exposed database ports).
3. Postgres and Redis are on a `backend` network unreachable from the host in production mode.
4. A container with a memory limit gets killed when it tries to allocate beyond the limit.
5. Log files rotate at 10 MB.
6. A manual backup of the postgres volume produces a tar file you can restore.

Commit:

```bash
git add .
git commit -m "feat: dev and prod compose split with networks and limits"
```

---

## What You Learned

- Override files split dev and prod without copy-pasting.
- Named networks isolate services.
- Volume backups work via a one-off container.
- Resource limits prevent runaway processes.
- Log rotation is mandatory for long-lived containers.

Tomorrow: Nginx reverse proxy, TLS, and the front door to your app.
