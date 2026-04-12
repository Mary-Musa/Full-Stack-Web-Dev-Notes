# Week 25, Day 2: Docker Compose

By the end of today, one `docker-compose up` starts your API, notification service, Next.js shop, Postgres, and Redis -- all networked together -- with one command. This is the setup that gives new developers a working dev environment in 60 seconds.

**Prior concepts:** Docker basics (Week 25 Day 1).

**Estimated time:** 3 hours

---

## Why Compose

Running each container with `docker run` gets tedious fast. Five services means five commands, five sets of flags, five networks to wire manually. `docker-compose` replaces all of that with a single YAML file.

Compose is for development and small production. Real production with many hosts uses Kubernetes, Nomad, or ECS. Compose is the "one laptop or one server" option, and that is exactly what most small Kenyan SMEs need.

---

## Your First `docker-compose.yml`

At the repo root:

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: crm_user
      POSTGRES_PASSWORD: crm_dev_password
      POSTGRES_DB: crm
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/schema.sql:/docker-entrypoint-initdb.d/schema.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U crm_user -d crm"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  api:
    build: ./api
    ports:
      - "5000:5000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      NODE_ENV: development
      DB_HOST: postgres
      DB_PORT: 5432
      DB_USER: crm_user
      DB_PASSWORD: crm_dev_password
      DB_NAME: crm
      REDIS_URL: redis://redis:6379
      CRM_SERVER_URL: http://api:5000
    env_file:
      - ./api/.env

  notifications:
    build: ./notification-service
    depends_on:
      - postgres
      - redis
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_USER: crm_user
      DB_PASSWORD: crm_dev_password
      DB_NAME: crm
      REDIS_URL: redis://redis:6379
    env_file:
      - ./notification-service/.env

  shop:
    build: ./shop
    ports:
      - "3000:3000"
    depends_on:
      - api
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_USER: crm_user
      DB_PASSWORD: crm_dev_password
      DB_NAME: crm
      CRM_SERVER_URL: http://api:5000

volumes:
  postgres_data:
  redis_data:
```

Five services, two persistent volumes. Walk through the interesting parts.

---

## What The Services Do

**`postgres`** runs the official Postgres 16 Alpine image. On first start it runs `schema.sql` (thanks to the `docker-entrypoint-initdb.d` magic). Data persists in the named volume `postgres_data` so restarting the container does not lose your data.

**`redis`** runs Redis 7 Alpine. Same pattern -- data persists to a volume.

**`api`** builds from `./api/Dockerfile`. It depends on Postgres and Redis being *healthy* (not just started -- healthy means the healthcheck passes). It references Postgres and Redis by their service names (`postgres`, `redis`) because Docker's built-in DNS resolves those to the right containers.

**`notifications`** builds from `./notification-service/Dockerfile`. Depends on Postgres and Redis. No exposed port because it is not public -- other services talk to it internally via `http://notifications:5010` if needed.

**`shop`** builds from `./shop/Dockerfile`. Depends on `api`. Port 3000 is exposed to the host.

---

## Networking

Compose creates a default network for the services. Every service gets a DNS name matching its service key (`postgres`, `redis`, `api`, `notifications`, `shop`). Services talk to each other by name, not by IP.

This is why `DB_HOST: postgres` works -- inside the compose network, `postgres` resolves to the Postgres container's IP.

---

## Volumes vs Bind Mounts

Two kinds of persistent storage:

- **Named volumes** (`postgres_data`, `redis_data`): Docker manages the storage. Volumes survive container restarts. Use for database data.
- **Bind mounts** (`./db/schema.sql:/docker-entrypoint-initdb.d/schema.sql`): mount a host file/directory directly. Use for dev-time code syncing and for loading init files.

In dev mode you often bind-mount source code so changes reflect without rebuilds:

```yaml
api:
  volumes:
    - ./api:/app
    - /app/node_modules    # don't shadow container's node_modules
  command: npm run dev
```

The `/app/node_modules` trick: you want the host's source code synced, but not the `node_modules` (container's and host's might differ). The anonymous volume shadows the bind mount for that subdirectory. A small but important trick.

---

## Starting Everything

```bash
docker-compose up -d
```

`-d` runs in the background (detached). Without `-d` you see all logs interleaved in your terminal.

To see logs:

```bash
docker-compose logs -f api
docker-compose logs -f    # all services
```

To stop:

```bash
docker-compose down
```

`down` stops and removes containers but keeps volumes. To remove volumes too:

```bash
docker-compose down -v
```

Only do that when you want a clean slate.

---

## Common Compose Commands

```bash
docker-compose up -d              # start in background
docker-compose ps                 # list services and their state
docker-compose logs -f shop       # follow one service's logs
docker-compose exec api sh        # shell into a running container
docker-compose restart api        # restart one service
docker-compose build api          # rebuild one service's image
docker-compose down               # stop all
docker-compose down -v            # stop all + remove volumes
```

`exec` is the most useful during debugging. Shell into the API container and run commands inside it:

```bash
docker-compose exec api sh
/app # node --version
/app # ls node_modules
/app # env | grep DB
```

---

## Environment Variables

Three places env vars come from, in priority:

1. `environment:` in the compose file -- explicit overrides.
2. `env_file:` in the compose file -- from a `.env` file.
3. A `.env` file at the compose file's directory -- compose auto-loads this for variable substitution.

For secrets, use `.env` at the repo root (git-ignored) and reference the keys in compose:

```yaml
api:
  environment:
    META_ACCESS_TOKEN: ${META_ACCESS_TOKEN}
```

Compose replaces `${META_ACCESS_TOKEN}` with the value from the host `.env` at startup. Never commit secrets; commit `.env.example` listing the keys.

---

## Healthchecks

The `depends_on: condition: service_healthy` is important. Without it, the API tries to start before Postgres is ready and crashes. With it, compose waits for Postgres's healthcheck to pass first.

Every service should have a healthcheck. For Node apps, a simple curl of `/health`:

```yaml
api:
  healthcheck:
    test: ["CMD", "wget", "--quiet", "--tries=1", "-O", "-", "http://localhost:5000/health"]
    interval: 10s
    timeout: 3s
    retries: 3
    start_period: 10s
```

`start_period` is a grace period at startup; healthchecks during this time do not count as failures.

---

## Checkpoint

1. `docker-compose up -d` starts five containers.
2. `docker-compose ps` shows them all running and healthy.
3. `curl http://localhost:5000/health` returns 200.
4. `curl http://localhost:3000` returns the shop home page.
5. Placing an order triggers a job; `docker-compose logs -f notifications` shows the worker process it.
6. `docker-compose down` stops everything; `docker-compose up -d` brings it back with data intact.
7. `docker-compose exec postgres psql -U crm_user -d crm -c "SELECT count(*) FROM orders"` returns a count.

Commit:

```bash
git add docker-compose.yml
git commit -m "feat: docker-compose for full local stack"
```

---

## What You Learned

- Compose runs multiple containers with one file.
- Service names are DNS names inside the compose network.
- Named volumes persist data; bind mounts sync host files.
- `depends_on: condition: service_healthy` waits for readiness.
- A handful of `docker-compose` subcommands covers 90% of dev workflow.

Tomorrow: volumes, env var strategies, and more realistic dev setups.
