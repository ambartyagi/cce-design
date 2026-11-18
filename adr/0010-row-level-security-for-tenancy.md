# 0010. Postgres RLS + session variable as tenancy enforcement

Date: 2026-08-10
Status: Accepted — implemented in Phase 0 (migration 0001, `cce.db.tenant_session`)

## Context
Multi-tenancy is the design's hardest claim to get wrong quietly — an
application-layer-only `WHERE tenant_id = ?` filter is one missed clause away
from a cross-tenant leak, and that bug class is invisible until it's a
breach.

## Decision
Defense in depth: every tenant-facing DB session sets
`app.tenant_id` (transaction-local, via `set_config(..., true)`); Postgres
Row-Level Security policies enforce the match unconditionally, independent of
application code. The `cce_app` role does not own the tables, so policies
apply even to bugs in `cce_app`'s own queries. `tenants`/`api_keys` get RLS
enabled with **no policy** — `cce_app` is denied entirely; only the admin
(superuser) engine touches them.

## Alternatives considered
- **Application-layer filtering only** — rejected: exactly the single point
  of failure this decision exists to eliminate.
- **Separate database/schema per tenant** — rejected: doesn't fit the
  "shared database, partitioned by client" requirement from the original
  EazeWork-style brief, and doesn't scale operationally to many tenants on
  one Mini.

## Consequences
The walking skeleton's tenant-B isolation checks (in
`scripts/walking_skeleton.py`) are a direct test of this ADR, and pass. This
is also the enforcement layer the Design Review's threat-model section
(poisoning, cross-tenant leakage) assumes is already solid — everything else
in the threat model is *additional* to this, not instead of it.
