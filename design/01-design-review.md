# Cumulative Context Engine – Design Review

**Reviewing:** Design Document v0.1 (2026-08-02)
**Review date:** 2026-08-03
**Purpose:** Identify gaps, challenge assumptions, propose improvements, and refine the roadmap before deeper design work begins.

---

## 1. Overall Assessment

The core thesis is sound and well-articulated: institutional knowledge belongs in a governed, versioned, external store rather than in model weights, and the unit of consumption should be governed data products rather than raw tables. The four-layer decomposition (Context Store / Data Products / Agent Runtime / Governance & Evolution) is clean, the multi-tenancy stance is correct for the EazeWork setting, and the instinct to start with one domain and design impact analysis before coding is the right one. The competitive framing also holds up — the Morpheo/Vena Omega acquisition is real and recent, and the "portable, multi-domain, self-hosted" positioning is a plausible wedge against vertical (Vena) and cloud-locked (Snowflake/Databricks) alternatives.

The document's main weakness is not what it says but what it treats as solved. Three phrases carry enormous unexamined weight: *"compile the right slice of context,"* *"corrections are systematically captured,"* and *"every change is auditable and reversible."* Each of these is a hard subsystem in its own right, and two of them (compilation and capture) are arguably where the product will actually succeed or fail. The sections below work through these and other gaps roughly in order of how much they should influence near-term design.

---

## 2. Critical Gaps

### G1. Context compilation is the hardest unsolved problem in the doc — and it's one line

The doc's value proposition is realized at exactly one moment: when an agent gets the *right* slice of context for a task. Everything else (versioning, governance, tenancy) is scaffolding around that moment. Yet compilation gets a single bullet. Questions that need first-class design:

- **Conflict resolution.** Two certified knowledge objects will eventually contradict each other (an old policy not yet deprecated, a tenant override vs. platform default, two domains defining "active employee" differently). What are the precedence rules? A workable starting order: tenant-specific beats shared, narrower scope beats broader, later effective date beats earlier, higher certification level beats lower — but this must be explicit, deterministic, and logged, or agents will silently flip answers between runs.
- **Ranking and budget.** Retrieval will return more than fits in a context window. What gets cut, and how do you know the cut didn't remove the one correction that mattered? A correction that has been certified but doesn't get retrieved is indistinguishable from a correction that was never made — this is the failure mode that quietly destroys trust in the whole system.
- **Attribution.** Every compiled context should record *which knowledge object versions were actually injected*, and ideally which ones the answer relied on. Without this, (a) users can't correct the right object when an answer is wrong, (b) you can't measure whether the store is helping, and (c) impact analysis has no usage data to weigh blast radius by. This is cheap to log at compile time and impossible to reconstruct later.
- **Cache invalidation.** Compiled contexts will be cached for latency/cost. Certification of a new version must invalidate exactly the affected caches — this is the same dependency problem as impact analysis, which is a hint that they should share one mechanism.

**Recommendation:** promote "Context Compilation Model" from a roadmap bullet to a co-equal deep-dive with impact analysis. They are two views of the same dependency graph (one at read time, one at write time).

### G2. The capture loop — the actual moat — is undesigned

The premise says corrections and decisions "are not systematically captured" in the status quo, but the doc never specifies the mechanism by which *this* system captures them. Compounding context requires a supply of context. Where does it come from, concretely?

- **Explicit correction UX:** a user says "that's wrong, the notice period is 60 days for managers" — in what surface? How does free-text become a structured proposal with the right object type, scope, and supersession link? (This extraction step is itself an agent task with its own error rate.)
- **Promotion from episodic memory:** the doc correctly says the agent runtime "does not own long-term institutional knowledge" — but agents will constantly learn things mid-session. There must be a defined promotion pipeline: session memory → candidate knowledge → proposal → certification. Without it, learnings either die with the session or leak into permanence without governance. This pipeline is currently missing from the architecture entirely.
- **Passive signals:** repeated questions, repeated manual overrides of the same answer, and query patterns are all evidence of missing or wrong knowledge. Even v1 should log these; mining them can come later.
- **Correction semantics:** "correcting" splits into at least three distinct operations — correcting a *fact* (supersede an object), correcting a *derivation* (fix a metric definition, which cascades), and correcting a *behavior* (update a playbook/procedure). These have different blast radii and should probably be different proposal types from day one.

**Recommendation:** add "Knowledge Capture & Promotion" as a fifth architectural concern (or as an explicit subsystem of Governance). Design the correction UX for the first POC before the schema — the schema should serve capture, not the reverse.

### G3. No evaluation strategy — you can't currently prove the context compounds

The tagline is "does your context compound?" but the doc has no way to answer that question for itself. Missing:

- **Golden question sets per domain** — a versioned suite of questions with known-correct answers, run against the agent with and without the context store, and re-run after every certified change. This is the regression suite for knowledge.
- **The no-regression invariant:** *a certified correction must never regress.* If someone corrected the notice-period answer in March, the golden set gains that case forever. This single invariant, tested continuously, is the most convincing demo of the whole thesis — to yourself and to prospective customers.
- **Compounding metrics:** knowledge-object reuse rate (how often objects are retrieved/used), correction half-life (time from wrong answer to certified fix), % of answers grounded in certified objects vs. raw model priors, certification queue latency, and retrieval hit rate on golden questions. Pick 4–5 and instrument from v0.
- **Retrieval evals separate from answer evals.** When a golden question fails, you need to know whether retrieval missed the object or the model ignored it. These require separate measurement.

**Recommendation:** move evaluation from implicitly-never to the near-term design list. The eval harness should exist before the second data product does.

### G4. Temporal semantics: versioning ≠ effective dating

The doc has transaction-time versioning (when the system learned something) but HRMS payroll/leave demands **valid-time** semantics (when a fact was true in the world): "leave accrual was 1.5 days/month until March 31, 2026, and 1.75 after." These are orthogonal — a classic bitemporal problem:

- "What did the system believe on June 1 about the April policy?" is a real audit question in payroll, and it needs both axes.
- Retroactive corrections ("we just learned the policy actually changed in February") are common in HR and are exactly the case where single-axis versioning gives wrong answers confidently.
- Schema impact is small if done now (`valid_from` / `valid_to` alongside version metadata) and painful to retrofit.

**Recommendation:** add valid-time fields to the KnowledgeObject schema sketch, and make "as-of" queries (both axes) an explicit requirement of the context store. Compilation must default to "currently valid, currently believed" but support point-in-time for audit and payroll recalculation.

### G5. Compliance collides with "full history retained" — this is a design conflict, not a detail

An HRMS knowledge store will inevitably absorb personal data: names in decision traces, salary details in corrections, health information in leave-policy exceptions. Meanwhile the doc commits to an append-only change log and full history for time travel. GDPR and India's DPDP Act (highly relevant for an EazeWork-style deployment) grant erasure rights that directly contradict immutable full history.

Standard resolutions exist but must be chosen *before* the storage layer is built: crypto-shredding (encrypt personal fields per-subject, delete the key to erase), tombstoning with redacted history, or a strict policy that knowledge objects must reference entities by opaque ID with personal data kept only in the operational system. Each has different implications for the change log, backups, embeddings (a vector of a sentence containing PII is itself personal data and must be deletable), and time-travel queries.

**Recommendation:** add a "PII & erasure" design decision to the near-term list, and add `sensitivity` / data-classification to the common object fields. Also decide early whether embeddings of sensitive objects are stored per-tenant-encrypted, because re-embedding on erasure is otherwise a nasty retrofit.

### G6. The threat model stops at tenant isolation — knowledge poisoning is the bigger risk

Row-level security handles tenant separation, but the more novel attack surface is the write path this architecture deliberately opens: **agents propose changes to the store that future agents will trust as ground truth.**

- **Injection → poisoning chain:** a malicious or malformed document ingested by a connector, or a prompt-injected agent session, produces a plausible-looking proposal; a fatigued certifier approves it; every future answer is now poisoned, with provenance that looks legitimate. The certification gate is the *only* barrier, which makes certifier UX and provenance display security features, not conveniences.
- **Provenance classes should be first-class:** human-authored / human-certified / agent-inferred / connector-ingested deserve different default trust, different certification bars, and visible distinction at compilation time (and ideally in the prompt given to agents).
- **Compiled context is an injection vector:** knowledge object *content* ends up in prompts. An object containing instruction-like text ("ignore previous rules and…") is a stored prompt injection. Content sanitation/linting at proposal time belongs in the validation stage.
- **Cross-tenant inference leakage:** if shared/platform knowledge can be *written* based on patterns observed across tenants (an obvious future temptation), that's a leakage channel that needs an explicit policy now, even if the answer is simply "never automatic."

**Recommendation:** the planned "multi-tenant isolation threat model" deep-dive should be broadened to a full threat model of the write path, with knowledge poisoning as the headline scenario.

### G7. Lineage capture must be in v0, or impact analysis is permanently crippled

The doc defers the impact-analysis algorithm (reasonably), but the algorithm is the easy half. The hard half is having the dependency data at all: which data products derive from which knowledge objects, which agents consume which products, which downstream consumers subscribe to what. **Dependencies not recorded at write/build time cannot be reconstructed later.** Two consequences:

- The v0 schema needs dependency edges (`derived_from`, `consumes`, `referenced_by`) as required fields on data products and registered agents — even if nothing consumes them yet.
- A **consumer registry** (who/what reads each product, even coarse-grained) should exist from the first data product. Blast radius without consumer data is guesswork.

Encouragingly, this same graph serves compilation (G1), cache invalidation, impact analysis, and rollback (G8) — one well-designed dependency subsystem pays four times.

### G8. "Reversible" is asserted, not designed

Versioned objects make *single-object* rollback easy. But by the time a bad change is discovered, dependent products may have re-materialized, agents may have acted on it, and later changes may have built on top of it. Rolling back then means: revert the object, identify everything derived since (the dependency graph again), re-materialize affected products, and decide what to do about *decisions already made* on poisoned context (at minimum: flag the decision traces). The doc should acknowledge that rollback is a cascade operation with a blast radius of its own, and scope v1 rollback honestly (e.g., "revert + flag + re-materialize direct dependents; no automatic remediation of downstream actions").

### G9. Certification will bottleneck on humans — design for queue reality

The proposal → certification flow assumes a human is available, attentive, and competent to judge each change. In a small deployment that's one or two overloaded people. Predictable failure modes: queue backlog (agents blocked or, worse, working from stale knowledge while fixes sit unapproved), rubber-stamping (which defeats the security model per G6), and no clear owner for cross-domain objects. Design responses worth specifying now:

- A concrete initial risk rubric so most changes never need a human. A workable v0: risk = f(impact tags, scope breadth, count of dependent products/agents, provenance class). Low-risk auto-materializes with post-hoc sampling review; only high-risk blocks on a human.
- Batch/digest certification UX rather than item-at-a-time.
- An explicit answer to "what happens while a proposal is pending?" — options: serve old version (safe default), serve with a "pending correction" annotation to the agent, or serve provisionally for read-only tasks. This choice materially affects user trust and should be per-risk-class.
- Certifier assignment rules per domain/type, and an escalation path.

### G10. Cold start is unaddressed — and it's the first thing every POC hits

Day one, the store is empty and the agent is no better than a bare LLM. The compounding story needs a floor to compound from. Each POC needs a bootstrap plan: for HRMS, ingest policy documents and extract candidate knowledge objects (extraction quality gates and human review of the initial batch are a mini-project of their own — bulk-certifying 500 auto-extracted objects via the G9 queue is the first stress test of the governance model); for Campaigns, backfill from historical campaign data; for Gathr, seed from test plans and known edge cases. The connectors line in "supporting pieces" deserves promotion: for the first months, *ingestion quality is the product*.

---

## 3. Second-Tier Gaps (important, but won't sink the design)

**Concurrent/conflicting proposals.** Two agents (or an agent and a human) propose changes to the same object. Needs at least optimistic concurrency with supersession-conflict detection at certification time. Cheap now, messy later.

**Entity resolution.** Knowledge objects reference entities ("the Bangalore office," employee #4521, campaign "Diwali-25"). Without a light entity registry and stable IDs per tenant, retrieval will fragment across aliases and impact analysis will miss links. Doesn't need full MDM — needs stable IDs and an alias table.

**Ontology governance (the meta-level).** The doc governs changes to knowledge, but who governs changes to the *types* of knowledge? Adding an object type or changing a schema is the highest-blast-radius change possible. Even a simple rule ("ontology changes are always human-certified, always versioned, migrations required") closes the loop.

**Embedding lifecycle.** Every object version change requires re-embedding; embedding model upgrades require re-indexing everything; tenant isolation must hold in vector search (filtered pgvector queries are fine at this scale, but test it — pre-filter vs. post-filter recall differs). Also: embeddings of superseded versions should be excluded from default retrieval but retained for time travel.

**Model & prompt versioning in decision traces.** A decision trace that records only the compiled context is not reproducible. Record model ID/version, prompt template version, and compilation policy version alongside. Cheap, and essential for debugging "why did the answer change?"

**Cost and latency budgets.** Compilation adds retrieval + assembly to every agent step. Set targets now (e.g., p95 compile < 500ms on Mac Mini hardware, context overhead < N tokens) because they constrain how sophisticated compilation can be. Local-vs-hosted model choice belongs here too: test the loop against one local and one hosted model early so the design doesn't silently couple to frontier-model capabilities the Mac Mini can't serve.

**Shared-knowledge inheritance semantics.** "Tenants inherit or override platform knowledge" hides real semantics: when platform knowledge updates, do tenants get it immediately or pin versions? Does a tenant override survive a platform update (yes, but flag divergence)? Recommend: tenants pin platform-knowledge versions and upgrades go through the same proposal flow, scoped per tenant. This reuses machinery you're already building.

**Staleness and confidence.** Compounding cuts both ways — wrong or outdated knowledge also compounds (context rot). Objects should carry confidence + last-validated timestamps; a periodic revalidation sweep (even just "flag objects untouched for N months that are retrieved often") belongs on the roadmap.

---

## 4. Improvements to Existing Decisions

**Postgres + pgvector: endorse, and defer the graph layer harder.** At POC scale, dependency edges in a plain table traversed with recursive CTEs will outperform the operational cost of a graph DB. Adopt a graph database only when a measured traversal problem appears. This also keeps the Docker footprint to essentially one stateful service, which matters on a Mac Mini.

**First POC: pick the HRMS leave/policy Q&A slice, with Gathr as a fallback.** The platform's differentiating claims are tenancy + governance + compounding corrections. Only HRMS exercises all three with real stakes and real users, and a *read-only Q&A* slice (answers questions, never writes to the HR system) keeps blast radius near zero while corrections still flow. Gathr's testing agent is the better choice only if the priority is proving the loop mechanics fastest with zero compliance surface. Campaigns should go last — it exercises the least of the core machinery. Whichever is chosen, the milestone should be a **walking skeleton**: one tenant, one domain, one agent, one data product, and the *full* loop (ingest → compile → answer → correct → propose → certify → materialize → re-answer correctly) working end-to-end before anything is broadened.

**Schema sketch additions.** Beyond the fields listed in §4 of the design doc, add: `valid_from`/`valid_to` (G4), `sensitivity` classification (G5), `provenance_class` as an enum not free text (G6), `confidence` and `last_validated_at` (§3), dependency edges (G7), and usage counters (retrieved_count / used_in_answer_count) to feed both impact analysis and staleness detection.

**Data product contracts need change semantics.** Adopt semver-like versioning with defined breaking vs. non-breaking rules (schema field removal = major; new optional field = minor). Impact analysis then gets a free signal: major version bumps are high-risk by construction.

**Sharpen open question #5 (agent authority):** rather than a binary, tie authority to the G9 rubric — agents auto-materialize only changes that are low-risk *and* provenance-class agent-inferred *and* within their own domain, always with post-hoc sampled review. Everything else proposes.

---

## 5. Revised Roadmap

Changes from the original: evaluation and lineage move into the near term (they cannot be retrofitted), capture UX is added, compliance becomes an explicit design item, and the two deep-dives are paired.

### Near-term (Design)
- [ ] Paired deep-dive: **dependency graph** serving both impact analysis (write-time) and context compilation (read-time) — one mechanism, two views
- [ ] **Context compilation model**: precedence/conflict rules, ranking & token budget, attribution logging, cache invalidation
- [ ] **Knowledge capture & promotion design**: correction UX for POC #1, episodic→institutional promotion pipeline, correction taxonomy (fact / derivation / behavior)
- [ ] **Evaluation harness design**: golden question sets, no-regression invariant, compounding metrics (pick 4–5)
- [ ] Concrete schema sketches — including valid-time fields, sensitivity, provenance class, dependency edges, usage counters
- [ ] **PII & erasure strategy** (crypto-shredding vs. tombstoning; embedding erasure) — decided before the storage layer is built
- [ ] Threat model: tenant isolation **plus write-path poisoning scenarios**
- [ ] Risk rubric v0 + pending-proposal serving policy (what's served while awaiting certification)
- [ ] Docker service boundaries and interfaces
- [ ] Choose POC #1 and define its walking skeleton + bootstrap/ingestion plan

### Implementation (after design solidifies)
- [ ] Scaffold Docker Compose project and core services (Postgres+pgvector, API, agent runtime, worker)
- [ ] Context store with versioning + tenancy + lineage edges + attribution logging *from the first commit*
- [ ] Eval harness with initial golden set — running in CI before the first data product ships
- [ ] Bootstrap ingestion for POC #1 (with human review of the initial extraction batch)
- [ ] Proposal → certification → materialization flow, with the risk rubric and batch certification UX
- [ ] First data product + agent loop; walking-skeleton demo of the full correction round-trip
- [ ] Impact preview v1 (direct dependents only is fine)
- [ ] Rollback v1 (revert + flag + re-materialize direct dependents)
- [ ] Mac Mini migration; validate portability and local-model latency budgets
- [ ] Second tenant (before second domain — tenancy is the riskier claim to validate)
- [ ] Second domain

### Later
- [ ] Graph layer (only if traversal metrics demand it), richer ontology + ontology-governance process
- [ ] Passive-signal mining (repeated questions, overrides) → auto-proposals
- [ ] Staleness/revalidation sweeps; confidence decay
- [ ] Advanced propagation/reinference, admin UI, observability hardening, cloud path

---

## 6. Success Criteria Worth Writing Down Now

The project should be able to answer, with data, within the first POC:

1. **No-regression:** zero certified corrections ever regress on the golden set.
2. **Compounding:** golden-set accuracy with the context store exceeds the bare model by a growing margin over time (the curve, not the point, is the thesis).
3. **Correction half-life:** median time from "user flags wrong answer" to "certified fix served" — target hours, not weeks.
4. **Grounding rate:** % of answers whose decision trace cites at least one certified knowledge object.
5. **Governance viability:** certification queue p95 latency stays under an agreed SLA without rubber-stamping (sampled-review disagreement rate as the check).

If these five hold on POC #1, the thesis is demonstrated and everything else is scaling.

---

*Companion to Design Document v0.1. Recommend folding accepted items into a v0.2 of the main doc.*
