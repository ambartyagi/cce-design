# Cumulative Context Engine – Dockerized Deployment Design

**Version:** 0.1
**Date:** 2026-08-03
**Status:** Design (pre-scaffold)
**Companion docs:** Design Document v0.1, Design Review v0.1
**Locked inputs:** Python + FastAPI · Postgres + pgvector · hosted LLM API with local-ready provider abstraction · domain-neutral walking skeleton · Docker Compose on MacBook → Mac Mini → cloud

---

## 1. Goals and Constraints

This document specifies how the Cumulative Context Engine (CCE) is packaged, run, and operated as Docker services, such that the identical images move from a development MacBook to a Mac Mini "production" node, and later to cloud infrastructure with configuration-only changes wherever possible.

Constraints that shape every decision below:

- **Single-node first.** The Mac Mini is one machine. No orchestration cluster, no service mesh, no distributed anything. Every component must justify its memory footprint.
- **ARM now, x86 later.** Apple Silicon (arm64) on the Mini; almost certainly amd64 in cloud. All images must be multi-arch from day one.
- **One person operates this.** Upgrades, backups, and debugging must be doable in minutes over SSH. Boring beats clever.
- **The LLM call dominates latency.** Nothing in this design needs to be fast in the microseconds sense; it needs to be *observable* and *restartable*.
- **12-factor discipline throughout** — config in environment, logs to stdout, stateless services, backing state in Postgres only — because that discipline *is* the cloud migration path.

---

## 2. Service Topology

### 2.1 The core decision: modular monolith, one image, multiple entrypoints

The four architectural layers (context store, data products, agent runtime, governance) are **modules in one Python codebase**, packaged into **one Docker image**, run as **three processes** distinguished only by their start command:

| Service | Command | Role | Scaling/failure profile |
|---|---|---|---|
| `api` | `uvicorn cce.api:app` | REST API: context store CRUD, proposals/certification, compilation endpoint, data products, tenancy enforcement | Stateless; restart freely; scale horizontally later |
| `worker` | `python -m cce.worker` | Async jobs: embedding, materialization, impact analysis, ingestion, eval runs, scheduled sweeps | Stateless; at-least-once jobs; safe to kill mid-run |
| `agent-runtime` | `python -m cce.agent` | Agent session orchestration: task loop, tool calls, LLM calls, short-term session memory, promotion proposals | Stateless between sessions; session state in Postgres |

**Why not microservices per layer:** separate images multiply build time, version skew, and inter-service auth for zero benefit at this scale. **Why not one process:** the three have different failure and load profiles — a wedged agent session or a heavy embedding job must not take down the API — and separating them *as processes* now is what makes separating them *as deployables* trivial in cloud later. This is the standard modular-monolith bet, and the module boundaries inside the codebase (`cce.store`, `cce.products`, `cce.agents`, `cce.governance`, `cce.llm`) are the future service boundaries if ever needed.

### 2.2 Full service inventory

```
┌─────────────────────────────────────────────────────────────┐
│  Docker network: cce-internal (no host exposure)            │
│                                                             │
│   ┌────────┐   ┌────────┐   ┌───────────────┐               │
│   │  api   │   │ worker │   │ agent-runtime │   (1 image)   │
│   └───┬────┘   └───┬────┘   └──────┬────────┘               │
│       │            │               │                        │
│       └────────────┼───────────────┘                        │
│                    ▼                                        │
│           ┌─────────────────┐      ┌────────────────┐       │
│           │ postgres        │      │ ollama         │       │
│           │ (pgvector)      │      │ [profile:      │       │
│           │ + named volume  │      │  local-llm]    │       │
│           └─────────────────┘      └────────────────┘       │
│                                                             │
│   ┌─────────┐            ┌──────────────────────────┐       │
│   │ migrate │ (one-shot) │ grafana/loki/prometheus  │       │
│   └─────────┘            │ [profile: observability] │       │
│                          └──────────────────────────┘       │
└──────────────────┬──────────────────────────────────────────┘
                   │ 443/80 only
              ┌────┴────┐
              │  caddy  │  (reverse proxy, TLS)
              └─────────┘
                   ▲
        Hosted LLM API (egress only)
```

Services and their status:

- **`postgres`** — the *only* stateful service. Official `pgvector/pgvector:pg17` image (multi-arch). Holds everything: knowledge objects, versions, dependency edges, decision traces, job queue, agent sessions, eval results. One database, schema-per-concern.
- **`migrate`** — one-shot container (same app image, command `alembic upgrade head`). Compose dependency ordering makes `api`/`worker`/`agent-runtime` wait for its successful exit. This makes "schema is current" a deploy invariant rather than a hope.
- **`caddy`** — reverse proxy and TLS termination. Only container with published ports. Automatic HTTPS when a domain is pointed at it; internal CA for LAN-only use. Chosen over nginx/traefik for zero-config TLS and a config file short enough to memorize.
- **`ollama`** — *compose profile `local-llm`, off by default.* Present in the file from day one so the local-model path is a `--profile local-llm` flag, not a redesign. (See §6.)
- **`grafana` + `loki` + `prometheus`** — *compose profile `observability`, off by default.* See §9; not needed until the system has users beyond you.

**Deliberately absent:** Redis (see §2.3), a message broker, object storage (Postgres `bytea`/large-object is fine for POC-scale documents; MinIO or S3 slots in later behind the storage module), and any UI container (the admin UI is a later roadmap item).

### 2.3 Job queue: Postgres, not Redis

Background work (embedding, materialization, impact analysis, ingestion, evals) uses a **Postgres-backed job queue** (`procrastinate`, or a hand-rolled `SELECT … FOR UPDATE SKIP LOCKED` table if we want zero dependencies) rather than Celery+Redis. Reasoning: it removes an entire stateful service from the Mini; jobs become **transactional with the data they operate on** (a proposal and its impact-analysis job commit atomically — no "job fired but row not committed" races, which matters for governance integrity); and queue depth is inspectable with SQL. The known trade-off — Postgres queues degrade at high throughput — is irrelevant below thousands of jobs/minute, and the worker module isolates the choice if it ever needs revisiting.

---

## 3. Image and Build Strategy

**One `Dockerfile`, multi-stage, uv-based:**

1. **Stage `deps`:** `python:3.12-slim`, install `uv`, copy `pyproject.toml`/`uv.lock`, `uv sync --frozen --no-dev` into a venv. Layer-cached: dependency changes rebuild this; code changes don't.
2. **Stage `runtime`:** fresh `python:3.12-slim`, copy the venv and the `cce/` package. Non-root user (`USER cce`), `PYTHONUNBUFFERED=1`, no compilers or build tools present in the final image.

Practices locked in from the first build:

- **Multi-arch by default:** `docker buildx build --platform linux/arm64,linux/amd64`. Built and pushed to a registry (GHCR is the pragmatic default) even while the only consumer is the Mini — the registry is what makes MacBook→Mini deploys pull-based and cloud migration trivial.
- **Pinned bases by digest**, and `uv.lock` makes Python deps exactly reproducible. An image built today must be buildable identically in six months when debugging a decision trace (the review's reproducibility requirement extends to the runtime itself).
- **Image tags = git SHA** plus a moving `latest` for dev. The running system reports its own SHA via `/healthz` so "what's deployed on the Mini?" is one curl.
- **Healthchecks in the compose file, not the Dockerfile:** `api` exposes `/healthz` (checks DB connectivity) and `/readyz` (checks migrations current); `worker` and `agent-runtime` touch a heartbeat row that `/healthz` can report on. Compose `depends_on: condition: service_healthy` gives correct startup ordering.
- **Embedding model note:** if we later choose local embeddings (sentence-transformers) instead of hosted, model weights are **baked into a separate layer or mounted volume, never downloaded at container start** — the Mini must come up cleanly with no internet.

---

## 4. Compose File Architecture

Three files, standard Compose merge semantics:

| File | Purpose | Where used |
|---|---|---|
| `compose.yaml` | Canonical topology: all services, networks, volumes, profiles, healthchecks | Everywhere |
| `compose.override.yaml` | Dev conveniences: bind-mount source for hot-reload (`--reload`), exposed Postgres port, verbose logging | MacBook only (auto-merged by default) |
| `compose.prod.yaml` | Production posture: `restart: unless-stopped`, resource limits, log rotation, no source mounts, pull-only images | Mac Mini: `docker compose -f compose.yaml -f compose.prod.yaml up -d` |

Profiles: `local-llm` (ollama) and `observability` (grafana stack) as described above. The default profile is exactly the six-container core (api, worker, agent-runtime, postgres, migrate, caddy) — **~1.5GB RAM total**, leaving the Mini's headroom for Ollama later.

**Environment layering:** a committed `.env.example` documents every variable; real `.env` files are per-machine and never committed. All configuration is read through a single `pydantic-settings` `Settings` class — one place to see every knob, typed and validated at startup (a service with a missing/invalid setting fails fast at boot, not at first use).

What lives in env vs. database — the rule: **env holds deployment facts** (DB URL, LLM API keys, provider selection, log level, external base URL); **the database holds product facts** (tenants, domains, risk policies, model routing rules, feature flags per tenant). Tenants must never require a redeploy to add.

---

## 5. Data, Volumes, and Backup

- **One named volume:** `cce-pgdata`. Nothing else in the system may hold state — enforced culturally now, structurally later (cloud containers get read-only filesystems).
- **Backups are non-negotiable from week one** because this system's entire value proposition is accumulated state. Baseline: a `backup` one-shot service (same compose file) running `pg_dump -Fc`, invoked nightly by the Mini's `launchd`/cron, writing to a local backups directory **and** synced off-machine (`restic` to any S3-compatible or even iCloud Drive target). Retention: 7 daily, 4 weekly, 6 monthly.
- **Restore is rehearsed, not assumed:** a documented `make restore-check` that spins up a scratch Postgres container, restores last night's dump, and runs a row-count sanity script. Run monthly. An untested backup of a "cumulative" system is a time bomb.
- **Time-travel and the change log live inside Postgres** (append-only tables), so backup of one volume genuinely captures full system history — a deliberate simplification the single-store design buys us.
- **PII carve-out (from review G5):** if crypto-shredding is adopted, per-subject keys live in a separate small table with *different backup retention* than the main dump, so key deletion (erasure) isn't silently undone by a restore. This must be designed with the schema, not after the first backup exists.

---

## 6. LLM Provider Abstraction

The `cce.llm` module is the only code in the system permitted to know which model is serving. It exposes three capabilities, each independently routable:

| Capability | Interface | POC default | Local-ready path |
|---|---|---|---|
| Reasoning / agent turns | `complete(messages, tools) → response` | Hosted API (Anthropic) | Ollama (e.g. llama/qwen-class) |
| Structured extraction | `extract(text, schema) → validated object` | Hosted API | Ollama — quality gate applies |
| Embeddings | `embed(texts) → vectors` | Hosted embeddings *or* local sentence-transformers | Local |

Design rules:

- **Routing is configuration, not code:** a `model_routing` table maps (task class, domain, tenant?) → provider+model. The walking skeleton ships with one row per capability.
- **Every LLM call is recorded** in the decision-trace layer with provider, model ID, parameters, token counts, latency, and the routing rule that selected it. This serves reproducibility (review §3), cost accounting per tenant, and the eventual A/B of hosted-vs-local.
- **Embeddings are the sticky one.** Vectors are only comparable within one model, so: every embedding row stores its `embedding_model_id`; the index is per-model; switching models is an explicit re-embed migration job (worker), never a config flip. Choose the POC embedding model with the expectation of keeping it for months. A local model (e.g. a bge/gte-class model via sentence-transformers in the worker) is worth considering *from the start* precisely because it makes the vector store independent of any vendor.
- **Ollama sizing reality check:** the Mini needs ≥32GB unified memory to serve a usefully strong local model alongside the core stack; on a 16GB Mini, local-llm should be considered out of scope and hosted API planned as the standing configuration. This should be checked against the actual hardware before promising a local-first future.
- **Failure posture:** hosted API outages degrade the system to read-only-ish (retrieval and CRUD still work; agent answers and extraction queue up). The worker retries with backoff; the API returns honest 503s for agent endpoints rather than hallucinating a fallback.

---

## 7. Networking, Auth, and Security Posture

- **One internal Docker network;** only `caddy` publishes ports. Postgres is reachable exclusively by service name inside the network (dev override may expose 5432 on the MacBook only).
- **Remote access to the Mini: Tailscale on the host** (not in a container). The Mini needs zero public ports for POC use — you and any collaborators reach `https://cce.<tailnet>.ts.net` privately, and Caddy can still do TLS via Tailscale certs. Public exposure becomes a deliberate later decision, not a default.
- **Auth v0:** per-tenant API keys (hashed at rest), presented as bearer tokens; middleware resolves key → `tenant_id` and stamps it into request context. **Every DB query goes through a session that has `SET app.tenant_id`**, with Postgres row-level-security policies as the second enforcement layer (the review's defense-in-depth stance made concrete). JWT/OIDC slots in at the same middleware seam when there are human users beyond the operator.
- **Egress is small and known:** the LLM provider, the registry, backup target. Worth writing down now because cloud security groups will want exactly this list.
- **Secrets:** `.env` file with `0600` perms on the Mini for POC (accepted risk, single operator, disk is FileVault-encrypted); the design names the upgrade path — Docker/compose secrets first, cloud secret manager at migration. No secret ever appears in an image, a log line, or a decision trace (the trace records *that* a key was used, never the key).
- **Container hardening defaults:** non-root user, `read_only: true` filesystems with explicit `tmpfs` for scratch, `no-new-privileges`, resource limits in `compose.prod.yaml`. Cheap now, table stakes in any future security review.

---

## 8. Migrations and Schema Lifecycle

- **Alembic owns the schema.** No table is ever created by application code at runtime.
- The `migrate` one-shot service is the *only* thing that runs migrations, in every environment including dev — so a migration that fails is discovered on the MacBook, not the Mini.
- **Expand-and-contract discipline** (add column → backfill → switch readers → drop later) from the start. With versioned knowledge objects and append-only logs, destructive migrations are almost never justified, and this discipline is what makes zero-downtime cloud deploys possible later without relearning habits.
- **Migration + ontology governance (review §3):** changes to knowledge-object *types* are ordinary Alembic migrations plus a required entry in the change log — the meta-level governance rule ("ontology changes are always human-certified") rides the same deploy mechanism.

---

## 9. Observability

Layered, so the Mini runs light while nothing is lost:

1. **Structured JSON logs to stdout** (structlog), with `tenant_id`, `request_id`, `trace_id` on every line. Docker's `local` logging driver with rotation caps (`compose.prod.yaml`). This layer alone must be sufficient to debug the POC — grep over `docker compose logs` is the v0 observability product.
2. **App-level truth in Postgres:** decision traces, LLM call records, job outcomes, compilation attribution (which object versions entered which answer — review G1). These are *product features*, not ops telemetry, and they live with the data because they're queried like data ("which answers used object X v3?").
3. **Metrics endpoint from day one, dashboards later:** `api` exposes `/metrics` (Prometheus format: request rates, job queue depth, LLM latency/tokens by provider, compilation cache hit rate). The `observability` profile (Prometheus + Grafana + Loki) consumes it when enabled — on the Mini this costs ~700MB RAM, which is why it's opt-in.
4. **One external uptime check** (any free pinger hitting `/healthz` via the tailnet or a public endpoint if exposed) so that "the Mini silently died Friday night" is noticed Saturday, not Monday.

---

## 10. Environments and the Deploy Loop

| Environment | Hardware | Compose invocation | Purpose |
|---|---|---|---|
| dev | MacBook | `docker compose up` (override auto-applies; source bind-mounted, hot reload) | Daily development; full stack runs locally including Postgres |
| prod-local | Mac Mini | `compose.yaml` + `compose.prod.yaml`, images pulled from registry | The standing system accumulating real context |
| cloud (later) | managed | see §11 | Scale-out / multi-operator |

**Deploy to the Mini is pull-based and three steps:** build+push multi-arch from the MacBook (or CI later) → on the Mini (via `docker context use mini` over SSH/Tailscale, so it's runnable from the MacBook): `docker compose pull && docker compose up -d` → `migrate` runs, healthchecks gate the restart, `/healthz` confirms the new SHA. Rollback is the same command with the previous SHA tag. A `Makefile` wraps these so deploy day one is `make deploy`.

**What "prod" means here even though it's a Mini in a closet:** images are immutable and registry-pulled (never built on the Mini), state is in one backed-up volume, config is in one file, and every deploy is a recorded SHA. Those four properties are the actual content of "production-ready" — the cloud migration only changes *where*, not *how*.

---

## 11. Cloud Migration Path

The design goal is that migration is a **re-homing of the same images and env vars**, not a rewrite. Mapping, in the order things would actually move:

| Compose service | First cloud step (managed-simple) | Full step (Kubernetes, only if needed) |
|---|---|---|
| `postgres` | **Managed Postgres with pgvector** (RDS/Cloud SQL/Neon) — move this first; it derisks the only stateful piece | same |
| `api` | Cloud Run / App Runner / ECS Fargate service | Deployment + HPA |
| `worker` | Same runtime, worker command, min-instances 1 | Deployment |
| `agent-runtime` | Same runtime, separate service | Deployment |
| `migrate` | Release-phase job / one-shot task before rollout | Job / init hook |
| `caddy` | Replaced by the platform's load balancer + managed TLS | Ingress |
| backups | Provider snapshots + retained logical dumps | same |
| `.env` | Secret manager + service env config | Secrets/ConfigMaps |
| Tailscale | Stays useful for private access; or VPC + IAP | same |

Things this design already did that make the migration config-only: multi-arch images in a registry, stateless services, stdout logging, health endpoints, one-shot migrations, no local filesystem state, egress list known, RLS-based tenancy (works identically on managed Postgres). The honest deltas to plan for: Postgres connection pooling (add `pgbouncer` or use the provider's pooler — worth adding to compose *before* migrating to test with it), object storage if documents outgrow Postgres, and CI taking over image builds.

**A useful intermediate:** the walking skeleton's eval harness (review G3) doubles as the migration acceptance test — golden set green on the Mini, migrate, golden set green in cloud, done.

---

## 12. Walking Skeleton (domain-neutral)

The first running milestone, using a built-in `demo` domain (seeded fixtures, no external connectors), exercising every architectural claim end-to-end:

1. `POST /tenants` (operator) → tenant + API key.
2. Seed: a handful of knowledge objects and one data product with dependency edges, via `POST /objects`, `POST /products`.
3. `POST /agent/sessions` → ask a question → compilation endpoint assembles context (attribution recorded) → LLM answers → decision trace stored.
4. Submit a correction: `POST /proposals` (from the session or directly) → impact preview (`GET /proposals/{id}/impact`, direct dependents v1) → `POST /proposals/{id}/certify` → materialization job runs → dependent product re-materialized.
5. Re-ask the question → corrected answer → eval harness records the golden case → `make eval` goes green.
6. Second tenant created → same question → demonstrably isolated answer (RLS proof).

When all six steps pass on the MacBook and then on the Mini after `make deploy`, the deployment design is validated; everything after is capability, not plumbing.

---

## 13. Mac Mini Sizing

| Component | Steady RAM | Notes |
|---|---|---|
| postgres | 300–600MB | tune `shared_buffers` modestly; pgvector indexes in RAM |
| api / worker / agent-runtime | ~150–250MB each | Python + FastAPI baseline |
| caddy | ~30MB | |
| **Core total** | **~1.5GB** | comfortable on any Mini |
| + observability profile | +700MB | optional |
| + ollama (useful model) | +8–24GB | **only sensible on a 32GB+ Mini** |

CPU is a non-issue for the core (LLM latency dominates); local embeddings in the worker are the only CPU-noticeable load and run fine on Apple Silicon. Disk: Postgres growth at POC scale is trivial; backups dominate — budget 10× database size for retention.

---

## 14. Decisions Made Here (and their status)

| # | Decision | Status |
|---|---|---|
| D1 | Modular monolith: one image, three process entrypoints | Proposed — cheap to revisit per-service |
| D2 | Postgres-backed job queue; no Redis/broker | Proposed — isolated behind worker module |
| D3 | Compose profiles for `local-llm` and `observability` | Proposed |
| D4 | Caddy for proxy/TLS; Tailscale for private access; no public ports by default | Proposed |
| D5 | GHCR registry + multi-arch buildx + SHA tags from first build | Proposed |
| D6 | Alembic-only schema changes via one-shot `migrate` service | Proposed |
| D7 | Nightly `pg_dump` + restic offsite + monthly restore rehearsal | Proposed — **non-negotiable in spirit** |
| D8 | Env = deployment facts; DB = product facts (tenants never need redeploy) | Proposed |
| D9 | Embedding model recorded per-row; model switch = explicit re-embed migration | Proposed |
| D10 | RLS + `app.tenant_id` session variable as tenancy enforcement | Proposed — aligns with review threat model |

Open items deliberately *not* settled here: exact queue library (procrastinate vs. hand-rolled), hosted embedding model choice, GHCR vs. other registry, Mini RAM size (drives the local-llm answer), backup offsite target.

---

## 15. Next Steps

1. **Confirm the Mini's specs** (RAM decides the local-LLM stance now rather than later).
2. **Scaffold the repo** to this design: compose files, Dockerfile, `pydantic-settings` config, Alembic baseline, healthchecks, Makefile — the "design doc + scaffold" step deferred from this round.
3. **Schema deep-dive** (KnowledgeObject/DataProduct/ChangeEvent/dependency edges with the review's additions) — the first real Alembic migration.
4. **Walking skeleton implementation** against §12, eval harness included.
5. `make deploy` to the Mini; run §12 there; begin accumulating real context.

---

*Companion to Design Document v0.1 and Design Review v0.1.*
