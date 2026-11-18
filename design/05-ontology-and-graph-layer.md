# Cumulative Context Engine – Ontology & Graph Layer

**Version:** 0.1
**Date:** 2026-08-12
**Companion docs:** Design Document v0.1 · Design Review v0.1 · Docker Deployment Design v0.1 · Use Cases & Domain Config v0.1 · Reporting & Agent Archetypes v0.1
**Question addressed:** Are ontologies and graph-based knowledge systems the same kind of thing we're building? Should we add them, and how?

---

## 1. Short Answer

You are already building both — in deliberately lightweight form. The domain packs' object types with JSON Schemas *are* an ontology (types, properties, constraints); the `dependency_edges` table and supersession links *are* a knowledge graph (typed, directed relationships between versioned nodes). What the buzz refers to is richer versions of each: formal class hierarchies and typed relationships on the ontology side, and entity-centric multi-hop traversal on the graph side.

So the question is not "should we add a different kind of system" but "how far up each ladder should we climb, and when." The recommendation: climb deliberately — an **entity layer and richer edges now** (cheap, high value, already implied by the Review), **pack-defined relationship ontology next**, and **derived graph indexes / graph databases only when a measured need appears**. The one non-negotiable: whatever graph machinery arrives, **governance stays at the knowledge-object level** — the graph serves retrieval and impact analysis; it never becomes a side door around certification.

---

## 2. Mapping the Buzz to What We Have

Four distinct things travel under "ontologies and knowledge graphs." They are frequently conflated; they have very different costs.

**Formal ontologies (RDF/OWL/SHACL, description logic).** Class hierarchies, formally defined relationships, machine reasoning ("a Manager is-a Employee; leave rules for Employees therefore apply to Managers"). Powerful in domains needing strict logical inference (biomedicine, aerospace compliance); heavyweight to author and govern. **Relation to us:** our pack ontologies are the 20% that gives 80% — named types, schema-validated properties, explicit scoping. Full OWL reasoning is almost never the binding constraint in business context systems, because the LLM now supplies the flexible-inference layer that description logics used to be the only way to get. Verdict: adopt the *concepts* (type hierarchies, relationship constraints), skip the *stack* (triple stores, reasoners).

**Property-graph knowledge graphs (Neo4j-style).** Entities and typed relationships as first-class citizens; traversal queries ("employees → reporting chain → cost centers → policies"). **Relation to us:** our graph is currently thin — edges exist mainly for lineage/impact. The genuinely missing piece is the **entity layer**: knowledge objects talk *about* things (employees, entities, creators, cards, tax sections) but those things aren't nodes yet. This was already flagged in the Review (§3, entity resolution); the graph framing makes it concrete.

**GraphRAG-style derived graphs.** An LLM extracts an entity/claim graph from a corpus; retrieval traverses it (and community summaries of it) instead of relying on vector similarity alone. Strong for multi-hop and "tell me everything about X" questions over large document sets. **Relation to us:** this is a *retrieval index technology*, not a knowledge store. Critically, auto-extracted triples are exactly the kind of ungoverned "truth" our architecture exists to prevent — so if adopted, the extracted graph must be a **derived, regenerable, uncertified index over certified objects**, never a peer of them (see §4).

**Semantic layers in data platforms (metric ontologies, Palantir-style object models).** Certified business definitions bound to data. **Relation to us:** this is nearly isomorphic to our MetricDefinition objects + data products. We're building the portable, multi-domain version of it — validation that the design is pointed where the industry is going, not a gap.

---

## 3. What Graph Structure Concretely Buys Us

Grounded in the use-case catalog, in descending order of value:

1. **Better impact analysis.** Already graph-shaped; richer edge types (`applies_to`, `derived_from`, `interprets`, `depends_on`) with kind-aware traversal turn "direct dependents" into a real blast radius. The taxation agent's rule-change → stale-interpretations flow (Reporting doc §4) *requires* this.
2. **Multi-hop compilation.** "What applies to a manager in the Bangalore entity?" needs scope traversal (employee → grade/entity → applicable rules), not keyword luck. One-hop edge expansion after retrieval fixes the most common retrieval miss: the certified correction whose wording doesn't match the question.
3. **Conflict detection.** Two active objects claiming the same entity/scope/period become *detectable* when scope is edges to entity nodes rather than prose. This feeds the compilation precedence problem (Review G1) with data instead of guesswork.
4. **Entity-centric answers.** "Everything we know about creator X / card Y / the UAE entity" — a traversal, impossible with retrieval-by-similarity alone.
5. **Deduplicated aboutness.** Alias resolution ("Bangalore office" = "BLR entity" = entity #E-14) stops knowledge from fragmenting across names — the silent killer of compounding in every domain.

---

## 4. The Architecture Rule: Two Layers, One Source of Truth

The design's central claim is that knowledge is governed, versioned, and certified. Graph techniques threaten that claim only if extracted structure is allowed to *be* knowledge. Hence one rule, stated once and enforced everywhere:

**Layer 1 — Certified knowledge (source of truth).** Knowledge objects, entities, and *declared* edges. Everything here is versioned, provenance-classed, and flows through proposals. Small, curated, compounding.

**Layer 2 — Derived indexes (regenerable, uncertified).** Embeddings (already), and later: LLM-extracted claim/entity graphs, community summaries, materialized traversal closures. Rebuildable from Layer 1 + source documents at any time; carries no authority; retrieval may *use* it, but answers cite Layer-1 objects only, and nothing in Layer 2 can change an answer's certification status.

Embeddings already follow this pattern (a vector is a derived, regenerable index of an object). GraphRAG-style extraction, if and when adopted, slots into the same box — which means the governance model needs *zero* changes to absorb it.

---

## 5. Staged Adoption Plan

### Stage 1 — Entity layer + richer edges (next schema migration; days, not weeks)

- New tables: `entities` (id, tenant_id, domain, type, canonical_name, attrs JSONB) and `entity_aliases` (alias → entity). Entity types are declared in domain packs like object types.
- Extend `dependency_edges` semantics: object→object kinds (`derived_from`, `interprets`, `supersedes` — making the existing implicit link explicit) and object→entity kinds (`applies_to`, `about`, `scoped_to`).
- Proposals gain edge payloads: creating/superseding an object can declare its edges; certification materializes them. Edges are Layer 1 — they go through governance like everything else.
- Compilation v1.1: after retrieval, expand one hop along `applies_to`/`interprets`/`derived_from` edges, add neighbors to the candidate set before ranking; attribution records which objects arrived via expansion.
- Impact preview v1.1: recursive CTE over edges with kind filters and depth 2–3, replacing the direct-dependents-only query.

This stage is pure Postgres — no new services, no new images; it closes the Review's entity-resolution gap and materially improves the two subsystems the Review called co-equal deep-dives (compilation and impact).

### Stage 2 — Relationship ontology in packs + conflict detection

- `ontology/relationships.yaml` per pack: which edge kinds are valid between which types (`PolicyRule --applies_to--> OrgUnit|Grade|Country`), with cardinality hints. Validation at proposal time; violations are certification blockers. This is "ontology" in the useful sense — constraints that catch errors, not a reasoner.
- Conflict detector (worker job): active objects of the same type sharing scope edges and overlapping valid-time windows → flagged pair queued for human resolution. The compilation precedence rules then consume the resolutions.
- Optional type hierarchies where a domain genuinely needs them (Gathr's card taxonomy; HRMS grade ladders) — as pack data, not platform code.

### Stage 3 — Derived graph indexes (only on measured need)

- GraphRAG-style extraction over ingested documents into Layer 2, primarily to *accelerate bootstrap*: extracted claims become proposal candidates for human certification (the Tenant Policy Bootstrap use case, UC-H6, gets a smarter front end) — extraction feeds governance rather than bypassing it.
- Community summaries / entity dossiers as cached Layer-2 artifacts for "everything about X" queries.
- Graph database (or Apache AGE inside Postgres, if Cypher syntax earns its keep) **only when** recursive-CTE traversals measurably fail — the deployment design's deferral stands, now with a concrete tripwire: p95 impact-analysis or compilation-expansion latency exceeding budget at real data volumes.

---

## 6. What We Deliberately Don't Do

No triple-store/RDF stack, no OWL reasoner, no ontology-first modeling phase (months of taxonomy workshops before the first correction loop runs is the classic way these projects die), no auto-materialized extracted knowledge, and no graph database purchase ahead of a measured traversal problem. The compounding loop stays the product; the graph makes the loop's two hardest subsystems (compilation, impact) better.

**Positioning takeaway:** the honest label for what we're building is a **governed knowledge graph with a compounding loop**. That phrase is also the answer to "how is this different from GraphRAG/Neo4j+LLM projects": those govern nothing — extraction *is* their truth; here, extraction only ever proposes, and certification decides.

---

## 7. Next Steps

1. Fold Stage 1 into the pending schema deep-dive (entities, aliases, edge kinds, edge-bearing proposals) — it should ship as migration 0002 on the walking skeleton.
2. Update compilation and impact-preview code paths per Stage 1 (small, contained changes to `compiler.py` and `store.py`).
3. Add 3–5 golden questions per domain that specifically require one-hop expansion (the eval harness should prove the graph earns its complexity).
4. Revisit Stage 3 only after a real domain has ingested real documents.

---

*Companion to the CCE design series.*
