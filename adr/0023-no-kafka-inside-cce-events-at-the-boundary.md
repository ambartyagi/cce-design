# 0023. No Kafka inside CCE; events at the boundary through an outbox

Date: 2026-10-12
Status: **Proposed** (analysis: [architecture/06-events-and-kafka.md](../architecture/06-events-and-kafka.md))

## Context

Kafka was suggested for parts of the system. CCE is one image and one database; work is
handed over through Postgres in the same transaction as the data (ADR-0002); tenancy and
erasure rely on Postgres; partners run the stack on small servers. Volumes are a weekly
export per company, a monthly platform release and thousands of questions a day.

## Decision

1. **No broker inside CCE.** Jobs stay in the Postgres queue. Add `LISTEN/NOTIFY` so the
   worker starts a job immediately instead of polling every 2 seconds.
2. **Outbox for outgoing events**: an ordered `events` table written in the same transaction as
   the change (question raised, sync applied, rule changed, release published), served per
   tenant as `GET /events?after=<id>` and optional webhooks.
3. **Optional `kafka-bridge` compose profile** (off by default, ADR-0003) for when EazeWork runs
   Kafka: it consumes configuration-change events and enqueues a `sync_requested` job (the event
   is a trigger; the sync still reads a full export), and can relay the outbox to their topic.

## Alternatives considered

- **Kafka as the job queue.** Rejected: loses same-transaction enqueue (needs an outbox anyway),
  adds a stateful service, and tenancy and erasure would have to be rebuilt outside Postgres.
- **Configuration change events as the data** (apply each event as a knowledge change).
  Rejected for now: missed or reordered events would leave knowledge wrong with no way to
  notice. A full snapshot stays the source of truth.

## Consequences

- Nothing new to operate unless a deployment opts into the bridge.
- Revisit when several separately deployed services consume CCE events, job volume reaches
  thousands per minute, consumers need long replay, or EazeWork requires Kafka. A
  single-binary Kafka-compatible broker (Redpanda) is the first step then; the outbox stays.
