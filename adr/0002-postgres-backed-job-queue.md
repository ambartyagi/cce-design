# 0002. Postgres-backed job queue; no Redis/broker

Date: 2026-08-10
Status: Accepted — implemented in Phase 0 (`cce.worker`, `SELECT ... FOR UPDATE SKIP LOCKED`)

## Context
Background work (embedding, materialization, impact analysis, ingestion,
evals) needs a queue. A governance system additionally needs jobs to be
**transactional with the data they operate on** — a proposal and its
impact-analysis job must not commit independently, or a job can fire against
a row that never actually committed.

## Decision
A Postgres-backed queue (`SKIP LOCKED` claim pattern), not Celery+Redis.

## Alternatives considered
- **Redis/Celery** — rejected: adds a stateful service to a single-node
  deployment for no atomicity benefit, and queue depth becomes invisible to
  plain SQL.
- **A managed queue service** — rejected: not portable to the Mac Mini; ties
  the design to a cloud provider before deployment even reaches cloud.

## Consequences
Job/data atomicity for free; queue depth inspectable with SQL; one fewer
stateful service to back up. Known trade-off (Postgres queues degrade at high
throughput) is irrelevant below thousands of jobs/minute — revisit only if
that changes.
