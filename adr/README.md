# Architecture Decision Records

One file per decision, numbered, never edited after acceptance — if a decision
changes, write a new ADR and mark the old one Superseded. This is the record
of *why*, which the design docs' prose narrative doesn't always preserve once
it's been revised a few times.

## Template

```markdown
# NNNN. Title

Date: YYYY-MM-DD
Status: Proposed | Accepted | Superseded by NNNN

## Context
What problem/tension forced a decision.

## Decision
What we chose.

## Alternatives considered
What else was on the table, and why not.

## Consequences
What this makes easier, what it makes harder, what it defers.
```

## Log

| # | Title | Status |
|---|---|---|
| [0001](0001-modular-monolith-one-image-three-entrypoints.md) | Modular monolith: one image, three process entrypoints | Accepted |
| [0002](0002-postgres-backed-job-queue.md) | Postgres-backed job queue; no Redis/broker | Accepted |
| [0003](0003-compose-profiles-for-optional-services.md) | Compose profiles for local-llm and observability | Accepted |
| [0004](0004-caddy-and-tailscale-for-access.md) | Caddy for proxy/TLS; Tailscale for private access | Accepted |
| [0005](0005-registry-based-multiarch-images.md) | Registry-pulled, multi-arch, SHA-tagged images | Proposed |
| [0006](0006-alembic-only-schema-changes.md) | Alembic-only schema changes via one-shot migrate service | Accepted |
| [0007](0007-backup-and-restore-rehearsal.md) | Nightly pg_dump + offsite sync + rehearsed monthly restore | Accepted — implemented 2026-10-14 |
| [0008](0008-env-vs-database-configuration-split.md) | Env holds deployment facts; DB holds product facts | Accepted |
| [0009](0009-embedding-model-recorded-per-row.md) | Embedding model recorded per-row; model switch is an explicit migration | Accepted |
| [0010](0010-row-level-security-for-tenancy.md) | Postgres RLS + session variable as tenancy enforcement | Accepted |
| [0011](0011-postgres-pgvector-over-graph-database.md) | Postgres+pgvector now; defer a graph database until measured need | Accepted |
| [0012](0012-domain-packs-as-configuration-not-code.md) | Domains are packs (config), never platform code changes | Accepted |
| [0013](0013-two-layer-knowledge-certified-vs-derived.md) | Two-layer knowledge model: certified truth vs. derived indexes | Accepted |
| [0014](0014-typed-decision-models-for-governance-gates.md) | Typed decision models (Jev) for governance gates | Accepted — implemented 2026-10-08 (migration 0003) |
| [0015](0015-platform-tenants-and-integrator-keys.md) | Platform tenants (shared read-only knowledge) and integrator keys | Accepted — implemented 2026-10-09 (migration 0005) |
| [0016](0016-local-embeddings-and-hybrid-retrieval.md) | Local embeddings (qwen3-embedding:0.6b, 1024d) and hybrid lexical+vector retrieval | Accepted — implemented 2026-10-09 (migration 0006) |
| [0017](0017-configuration-first-hrms-knowledge.md) | Configuration first: HRMS knowledge starts from the EazeWork export; ambiguities become questions for HR | Accepted — implemented 2026-10-10 |
| [0018](0018-eligibility-and-transient-asker-profile.md) | Rule eligibility evaluated against a transient, whitelisted, consented asker profile | Accepted — implemented 2026-10-10 (migration 0007) |
| [0019](0019-follow-ups-rewrite-not-recall.md) | Follow-up questions: the conversation rewrites the question, it is never a source | Accepted — implemented 2026-10-10 |
| [0020](0020-admin-corrections-by-cause.md) | Admin corrections of answers, recorded by cause (rule / misread / data) | Accepted — implemented 2026-10-10 (migration 0008) |
| [0021](0021-compiled-distribution-image.md) | Partners get a compiled distribution image (no Python source, per-recipient build id) | Accepted — implemented 2026-10-12 |
| [0022](0022-sync-runs-and-version-pinned-interpretations.md) | Sync runs, version-pinned interpretations, and platform releases | Accepted — implemented 2026-10-14 (migration 0009) |
| [0023](0023-no-kafka-inside-cce-events-at-the-boundary.md) | No Kafka inside CCE; events at the boundary through an outbox | Proposed |
| [0024](0024-people-roles-and-eazework-as-identity-provider.md) | People, roles, and EazeWork as the identity provider | Accepted — roles implemented 2026-10-15 (migration 0010); EazeWork sign-in pending |
| [0025](0025-sync-triggers-plans-and-metering.md) | Sync triggers, plans and metering | Proposed |
| [0026](0026-layered-settings-and-company-data-controls.md) | Layered settings, and company-level data controls instead of per-company backups | Accepted — settings implemented 2026-10-15; data controls pending |
| [0027](0027-ops-instance-bots-as-principals.md) | A second CCE instance runs the company; bots are principals with runs and decisions (amended 2026-10-17: platform/client tenants) | Proposed |
| [0028](0028-autonomy-ladder-per-action-type.md) | Bot autonomy is a per-action-type level, earned by record and lost automatically | Proposed |

Status reflects what's actually running. "Proposed" ADRs (0005, 0023, 0025) are
decided in design but not yet implemented in the Phase 0 skeleton — see
ROADMAP.md.
