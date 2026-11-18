# Cumulative Context Engine – Design Document

**Version:** 0.1 (Discussion Capture)  
**Date:** 2026-08-02  
**Status:** Pre-implementation design  
**Scope:** Architecture, principles, and roadmap for a portable, multi-tenant cumulative context + data products platform

---

## 1. Context

This document captures the design discussion around building a system inspired by Morpheo AI (acquired by Vena Solutions in July 2026 and integrated as Vena Omega™ – described as a “cumulative context engine”).

The original insight from Morpheo was:

> The model isn’t the moat. Does your context compound?

Foundation models are powerful but lack durable, organization-specific context. Enterprises struggle to make agentic AI reliable because:
- Data is fragmented
- Business meaning is tribal and evolving
- Corrections and decisions are not systematically captured
- Continuous model fine-tuning risks catastrophic forgetting and is hard to govern

The goal is to build a system that treats **organizational context as a first-class, compounding, governed asset** rather than something that must be re-derived or baked into model weights.

### Target Use Cases (POCs)

1. **HRMS** (EazeWork-style) – multi-module HR, payroll, leave, attendance, performance, multi-country. Strong need for policy context, definitions, approval history, and auditability.
2. **Influencer / Campaigns system** – performance metrics, creative effectiveness, audience definitions, historical campaign learnings.
3. **Gathr (card game)** – starting with testing + monitoring agents; later broader game knowledge (edge cases, balance, failure patterns).

The same platform should support all three as different **domains**, and must work in a **multi-tenant** setting (shared database, data partitioned by client) such as EazeWork.

### Deployment Intent

- Package as Docker services for portability
- Run first on MacBook → move to Mac Mini
- Keep a clean path to cloud (Kubernetes / managed services) later

---

## 2. Premise / Core Thesis

**Continuously training or fine-tuning a model on organizational data is not the primary way to accumulate institutional knowledge.**

| Approach | Knowledge lives in… | Update model | Risk profile |
|----------|---------------------|--------------|--------------|
| Continual fine-tuning | Model weights | Expensive, opaque, global | Catastrophic forgetting, hard to audit/correct, governance difficulty |
| Cumulative Context System | Explicit, versioned store | Surgical, versioned, reversible | Low – knowledge is inspectable and editable |

**Preferred architecture:**
- Keep the model as a general (or lightly adapted) reasoner
- Store facts, definitions, metrics, assumptions, decision traces, corrections, and relationships in a governed external store
- Compile the right slice of context at inference / task time
- Let agents propose updates; humans (or risk-based policy) certify high-impact changes
- Treat **data products** as the primary consumable unit for agents and humans

This yields compounding context, precise corrections, full provenance, and avoids the main failure modes of pure weight-based continual learning.

---

## 3. High-Level Architecture

Four cooperating layers:

1. **Context Store**  
   Persistent, versioned, multi-tenant, multi-domain institutional knowledge (definitions, metrics, relationships, decision traces, corrections, playbooks).

2. **Data Products**  
   Governed, self-describing, addressable units that package content + semantics + quality + lineage + access contracts. Agents prefer products over raw tables.

3. **Agent Runtime**  
   Orchestration, tool use, short-term memory, human-in-the-loop. Reads from the context store; does not own long-term institutional knowledge.

4. **Governance & Evolution**  
   Proposal → validation → impact/blast-radius analysis → certification → materialization → propagation → monitoring. Every change is auditable and reversible.

Supporting pieces: connectors/ingestion, API + auth, observability, optional admin UI.

---

## 4. Key Design Decisions So Far

### Context Store
- Hybrid storage: structured store (versioning, tenancy, precise lookup) + vector index (semantic retrieval) + optional graph layer + append-only change log.
- Every knowledge object and data product carries: `id`, `version`, `tenant_id`, `domain`, `type`, `status`, `content`, `provenance`, supersession links, scope, impact tags, validation status.
- Soft deprecation preferred over hard deletes. Full history retained for audit and time-travel queries.

### Safe Updates
- Changes are proposed (by agents or humans), never silently written as permanent truth.
- Risk-based certification gates.
- Impact / blast-radius preview before materialization.
- Dependent data products and agents are flagged for refresh after changes.

### Multi-Tenancy
- `tenant_id` is first-class on virtually every object.
- Isolation enforced at retrieval, search, and context compilation time (database row-level security + application checks).
- Support private tenant knowledge (default) and optional shared/platform knowledge that tenants can inherit or override.
- Agent sessions are bound to a tenant.

### Cross-Product / Domains
- Platform is domain-agnostic.
- Each product (HRMS, Campaigns, Gathr) is modeled as a `domain`.
- Domain-specific connectors, schemas, agents, and risk policies plug into the shared core.

### Deployment
- Docker-first service split so the same images run on laptop, Mac Mini, and later cloud.
- Configuration (tenants, domains, model endpoints, feature flags) externalized from images.

### Relationship to Model Training
- Institutional knowledge stays outside weights.
- Occasional fine-tuning on high-quality trajectories is still possible as a complement, not the primary knowledge store.
- This largely sidesteps catastrophic forgetting of organizational facts and rules.

---

## 5. Competitive Landscape (Summary)

Related categories in 2026:
- **Enterprise context layers**: Atlan, DataHub Cloud, Databricks Genie Ontology, Snowflake Horizon Context, AWS Context, Couchbase AI Data Plane, etc.
- **Autonomous data products**: Nextdata OS (closest conceptual overlap on the data-products side).
- **Agent memory platforms**: Mem0, Zep/Graphiti, Letta (more episodic/conversational than full business context).
- **Vertical**: Vena Omega (finance/FP&A cumulative context engine).

Differentiation opportunity: a lightweight, portable (Docker), multi-product, multi-tenant cumulative context + data products engine that emphasizes compounding knowledge, safe update workflows, and full ownership/control.

---

## 6. Open Design Questions

1. Primary store choice (Postgres + pgvector as pragmatic default vs early graph DB).
2. How rich the initial ontology/schema should be.
3. Risk classification rules for automatic vs human certification.
4. Shared vs private knowledge model and tenant opt-in/out.
5. Agent authority to auto-materialize low-risk changes.
6. Depth of observability and decision-trace logging from day one.
7. Exact impact-analysis / blast-radius algorithm (next deep-dive).

---

## 7. Future Steps / Roadmap

### Near-term (Design)
- [ ] Deep-dive: Impact analysis / blast-radius algorithm
- [ ] Concrete schema sketches (KnowledgeObject, DataProduct, ChangeEvent, Tenant, Domain)
- [ ] Multi-tenant isolation patterns and threat model
- [ ] Agent context compilation model (progressive disclosure, scoping)
- [ ] Docker service boundaries and interfaces
- [ ] Minimal viable domain model for the first POC (recommend starting with one domain)

### Implementation (after design solidifies)
- [ ] Scaffold Docker Compose project and core services
- [ ] Implement context store with versioning + tenancy
- [ ] Basic proposal → certification → materialization flow
- [ ] First data product + agent loop in one domain
- [ ] Impact preview v1
- [ ] Move to Mac Mini and validate portability
- [ ] Expand to second domain and multi-tenant scenarios

### Later
- [ ] Richer ontology and graph relationships
- [ ] Advanced propagation and reinference agents
- [ ] Observability, admin UI, and production hardening
- [ ] Cloud deployment path

---

## 8. Immediate Next Topic

**Impact Analysis Algorithm**

How the system determines, before a change is materialized:
- Which knowledge objects, data products, agents, and downstream consumers are affected
- What the expected blast radius is
- How to present this clearly for human certification
- How to keep the analysis itself efficient and reliable as the knowledge graph grows

This is a critical safety and usability component and should be designed carefully before coding.

---

*Document maintained as the living design reference for the project. Update as decisions are finalized.*