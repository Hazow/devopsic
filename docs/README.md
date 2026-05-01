# Local Developer Environment

## Prerequisites

- Docker 24+ or Podman 4+ with Compose plugin
- Git

## Quick start

```bash
# 1. Clone the repository
git clone <repo-url>
cd devopsic

# 2. Create your local .env from the example
cp .env.example .env
# Edit .env and set a real POSTGRES_PASSWORD

# 3. Start the full stack (detached)
docker compose up -d

# 4. Verify everything is up
docker compose ps
curl http://localhost:8080/healthz
# Expected: {"status":"ok","service":"api"}
```

The frontend is available at http://localhost:3000.

## Service ports

| Service | Port |
|---------|------|
| frontend | http://localhost:3000 |
| api | http://localhost:8080 |
| postgres | localhost:5432 (internal only) |
| redis | localhost:6379 (internal only) |

## API endpoints

```bash
# Health check
curl http://localhost:8080/healthz

# List all events
curl http://localhost:8080/events

# Purchase a ticket (queues the order)
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"user@example.com","quantity":2}'

# View processed orders
curl http://localhost:8080/tickets/orders
```

## Hot-reload

Frontend (`src/`) and API (`src/`) source directories are mounted into the running containers. Changes to source files are picked up automatically by nodemon — no restart needed.

Worker source is also mounted; nodemon restarts the worker process on file changes.

## Logs

```bash
# All services
docker compose logs -f

# Single service
docker compose logs -f api
docker compose logs -f worker
```

## Stop the stack

```bash
# Stop containers, keep volumes (data is preserved)
docker compose down

# Stop containers AND delete all volumes (wipes the database)
docker compose down -v
```

## Environment variables

| Variable | Description | Default |
|---|---|---|
| `POSTGRES_DB` | Database name | `ticketing` |
| `POSTGRES_USER` | Database user | `ticketing_user` |
| `POSTGRES_PASSWORD` | Database password | **must be set** |
| `POSTGRES_HOST` | Database hostname | `postgres` |
| `POSTGRES_PORT` | Database port | `5432` |
| `REDIS_HOST` | Redis hostname | `redis` |
| `REDIS_PORT` | Redis port | `6379` |
| `API_PORT` | API listen port | `8080` |
| `FRONTEND_PORT` | Frontend listen port | `3000` |
| `API_BASE_URL` | API URL visible to the browser | `http://localhost:8080` |
| `QUEUE_NAME` | Redis queue name for orders | `ticket_orders` |
| `NODE_ENV` | Node environment | `development` |

All variables are loaded from the `.env` file at the project root. Never commit `.env` — it is in `.gitignore`.

## Troubleshooting

**API fails to start:** Redis or PostgreSQL may not be ready yet. Wait 10–15 seconds and run `docker compose restart api`.

**Orders are not appearing in `GET /tickets/orders`:** Check the worker logs — `docker compose logs worker`. The worker must be connected to both Redis and PostgreSQL.

**Port already in use:** Change `API_PORT` or `FRONTEND_PORT` in `.env` and restart the stack.
