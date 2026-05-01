# Architecture — Secure Event Ticketing Platform

## Container vs Virtual Machine

| | Virtual Machine | Container |
|---|---|---|
| **Isolation** | Full OS per VM, hypervisor-level | Process-level, shared kernel |
| **Startup time** | Minutes | Seconds |
| **Image size** | GBs (full OS) | MBs (only app + runtime) |
| **Resource overhead** | High — each VM reserves CPU/RAM for a guest OS | Low — containers share the host kernel |
| **Portability** | Tied to hypervisor format (OVA, VMDK) | Runs identically on any OCI-compliant runtime |
| **Security boundary** | Strong (hardware virtualisation) | Weaker by default — mitigated via namespaces, seccomp, non-root users |
| **Best for** | Workloads requiring full OS isolation or legacy apps | Microservices, stateless apps, CI pipelines |

**Why containers for this project:** All five services are stateless Node.js processes or managed open-source databases. They start in seconds, fit in small images, and benefit from identical environments from developer laptop to Kubernetes cluster. The overhead of a full VM per service is not justified.

---

## Service Selection

| Service | Role | Why this service |
|---|---|---|
| **frontend** | Serves the web UI (Express.js, port 3000) | Single-page HTML + JS app; thin server that delivers static assets and proxies `/config` to resolve the API URL at runtime |
| **api** | REST API (Express.js, port 8080) | Handles ticket purchase requests, validates input, publishes orders to Redis queue, serves order history from PostgreSQL |
| **worker** | Background job processor (Node.js, no HTTP) | Decouples order processing from the HTTP request cycle — consumes from Redis queue and writes confirmed orders to PostgreSQL |
| **postgres** | Relational database (PostgreSQL 16) | Persistent, ACID-compliant storage for ticket orders; supports structured queries and indexing |
| **redis** | In-memory queue and cache (Redis 7) | Lightweight message broker between api and worker using `LPUSH`/`BRPOP`; avoids tight coupling and handles traffic spikes |

---

## Data Flow

```
Browser
  │
  ▼ HTTP :3000
┌─────────┐
│frontend │  serves static HTML/JS
└────┬────┘
     │ HTTP :8080
     ▼
┌─────────┐   LPUSH ticket_orders   ┌────────┐
│   api   │ ────────────────────── ▶│ redis  │
└────┬────┘                         └───┬────┘
     │                                  │ BRPOP
     │ SELECT / ◀──────────────┐   ┌───▼────┐
     ▼                         │   │ worker │
┌──────────┐                   │   └───┬────┘
│ postgres │ ◀─────────────────┘       │
└──────────┘      INSERT ticket_orders ┘
```

**Request lifecycle for a ticket purchase:**
1. Browser sends `POST /tickets/purchase` to api
2. api validates the request and pushes a JSON order onto the `ticket_orders` Redis queue
3. api immediately returns `202 Accepted` with an `orderId`
4. worker picks up the order via `BRPOP` (blocking pop), inserts a row into `ticket_orders` in PostgreSQL
5. Browser can poll `GET /tickets/orders` to see the processed order

---

## Network boundaries (production)

```
Internet ──▶ Ingress ──▶ frontend ──▶ api ──▶ postgres
                                        └──▶ redis ◀── worker ──▶ postgres
```

- Only `frontend` and `api` are reachable from outside the cluster (via Ingress)
- `postgres` and `redis` accept connections only from `api` and `worker`
- `worker` has no inbound network surface

---

## Technology stack

| Layer | Technology | Version |
|---|---|---|
| Runtime | Node.js | 20 LTS |
| API/Frontend framework | Express.js | 4.x |
| Database | PostgreSQL | 16 |
| Queue / cache | Redis | 7 |
| Container runtime (dev) | Docker / Podman | — |
| Orchestration (prod) | Kubernetes | — |
| CI/CD | GitHub Actions | — |
| Image scanning | Trivy | latest |

---

## Image tagging policy

| Environment | Tag format | Example |
|---|---|---|
| Production | `git-<sha>` (first 7 chars of commit SHA) | `devopsic_api:git-a3f9c12` |
| Pull request | `pr-<number>` | `devopsic_api:pr-42` |
| Local dev | `dev` (never pushed to registry) | — |

**Rules:**
- `latest` tag is never used in production or pushed to any registry
- Every production image is traceable to the exact commit that built it via the SHA tag
- Images are only pushed after Trivy scan passes (no CRITICAL vulnerabilities)

---

## Security scan findings (Trivy)

Scanned images: `devopsic_api:prod`, `devopsic_frontend:prod`, `devopsic_worker:prod`
Scan date: 2026-05-01 | Trivy version: 0.70.0

| Image | CRITICAL | HIGH | Notes |
|---|---|---|---|
| devopsic_api:prod | 0 | 11 | HIGH from `node-tar` (transitive dep via npm) |
| devopsic_frontend:prod | 0 | 11 | same |
| devopsic_worker:prod | 0 | 11 | same |

**Decision:** `node-tar` HIGH CVEs (CVE-2026-23950, CVE-2026-24842, CVE-2026-26960, CVE-2026-29786, CVE-2026-31802) are in npm's internal tooling package, not in any runtime code path. These packages are not reachable in production workloads. Accepted risk — will be resolved when upstream `node:20-alpine` base image updates npm's bundled tar dependency.

Full machine-readable report: `security/trivy-report.json`
