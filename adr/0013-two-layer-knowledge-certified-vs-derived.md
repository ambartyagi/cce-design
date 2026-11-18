# 0013. Two-layer knowledge model: certified truth vs. derived indexes

Date: 2026-08-20
Status: Accepted (Ontology & Graph Layer doc §4)

## Context
Graph/embedding techniques (GraphRAG-style extraction, community summaries,
auto-derived entity graphs) are valuable for retrieval, but auto-extracted
structure is exactly the kind of ungoverned "truth" the whole architecture
exists to prevent (Design Review, knowledge-poisoning threat model). The risk
is extracted structure quietly becoming a peer of certified knowledge.

## Decision
Two layers, one source of truth. **Layer 1** (certified): knowledge objects,
entities, and declared edges — versioned, provenance-classed, only enters via
proposals. **Layer 2** (derived): embeddings (already implemented this way),
and later any extracted graph/summary — regenerable from Layer 1 + source
documents at any time, carries no authority. Retrieval may use Layer 2;
**answers cite Layer 1 objects only**; nothing in Layer 2 can change an
answer's certification status.

## Alternatives considered
- **Let extraction write directly to the knowledge store** — rejected: this
  is precisely the "knowledge poisoning" attack surface the Design Review
  flags — a plausible-looking auto-extracted claim, approved by a fatigued
  certifier, becomes trusted ground truth.

## Consequences
GraphRAG-style extraction (Ontology doc, Stage 3) can be adopted later with
**zero governance changes** — it's just a new thing that lives in Layer 2 and
feeds proposals, same as the embedding pipeline already does. This ADR is
the reason ADR 0011 can defer a graph database without deferring the value of
graph-shaped retrieval.
