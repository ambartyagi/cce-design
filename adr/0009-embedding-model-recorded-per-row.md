# 0009. Embedding model recorded per-row; model switch is an explicit migration

Date: 2026-08-10
Status: Accepted (schema in place); embeddings provider is currently `none` — retrieval is lexical-only (Phase 0)

## Context
Vectors are only comparable within the model that produced them. If the
embedding model changes silently, similarity search silently degrades with
no error, no log line, nothing to grep for.

## Decision
Every `knowledge_objects` row carries `embedding_model`. Switching models is
an explicit worker job (bulk re-embed), never a config flip. Choose the first
real embeddings provider expecting to keep it for months.

## Alternatives considered
- **One global "current model" setting** — rejected: exactly the silent-
  degradation failure mode this ADR exists to prevent.

## Consequences
**Open decision (see STATUS.md):** hosted embeddings vs. local
(sentence-transformers, possibly via the exo cluster) — needs to be made once
before Phase 1's HRMS golden set is built, since re-embedding later is a real
migration, not a config change.
