# 0006. Alembic-only schema changes via one-shot migrate service

Date: 2026-08-10
Status: Accepted — implemented in Phase 0 (`migrations/`, `compose.yaml` `migrate` service)

## Context
No table should ever be created by application code at runtime — that path
makes "what's the current schema" a question you can only answer by reading
code, and it breaks the reproducibility a versioned knowledge store depends on.

## Decision
Alembic owns every schema change. A one-shot `migrate` container runs
`alembic upgrade head` before `api`/`worker` are allowed to start
(`depends_on: condition: service_completed_successfully`), in every
environment including dev — so a bad migration is caught on the MacBook, not
the Mini.

## Alternatives considered
- **`Base.metadata.create_all()` at app startup** — rejected: no history, no
  rollback, no dev/prod parity guarantee.

## Consequences
Expand-and-contract discipline (add → backfill → switch → drop) is the
expected pattern from day one — cheap now, necessary once zero-downtime cloud
deploys matter (Deployment doc §8). Ontology changes (new knowledge-object
types) ride this same mechanism per the Ontology doc's governance rule.
