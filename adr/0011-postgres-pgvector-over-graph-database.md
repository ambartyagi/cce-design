# 0011. Postgres+pgvector now; defer a graph database until measured need

Date: 2026-08-20
Status: Accepted (Ontology & Graph Layer doc §5, Stage 3)

## Context
"Ontologies and knowledge graphs" naturally raises "should we use Neo4j (or
similar)?" Dependency edges, entities, and multi-hop traversal are all
genuinely useful (see the Ontology doc), but a graph database is a second
stateful service on a single-node deployment, and recursive CTEs in Postgres
handle POC-scale traversal fine.

## Decision
Model entities and typed edges in Postgres (Stage 1/2 of the Ontology doc's
adoption plan: `entities`, `entity_aliases`, richer `dependency_edges` kinds,
recursive-CTE traversal for impact analysis and one-hop compilation
expansion). Adopt a graph database only past a concrete tripwire: p95
impact-analysis or compilation-expansion latency exceeding budget at real
data volumes.

## Alternatives considered
- **Adopt a graph DB now, anticipating scale** — rejected: no measured
  traversal problem exists yet; this is the same "don't build ahead of the
  gate" discipline as the rest of the roadmap.
- **RDF/OWL with a reasoner** — rejected outright, not deferred: full logical
  inference is almost never the binding constraint once an LLM supplies
  flexible inference; adopt ontology *concepts* (typed relationships,
  schema constraints), not the reasoner stack.

## Consequences
Governance stays simple (one database to secure, back up, and apply RLS to).
The two-layer knowledge rule (ADR 0013) is what makes it safe to later add
GraphRAG-style extraction without this decision needing to be revisited —
extraction becomes a Layer-2 index, not a reason to change the store.
