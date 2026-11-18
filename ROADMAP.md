# Roadmap

Five phases. The architecture doesn't change shape after Phase 0 — every later
phase adds capability onto the same compounding loop (ask → answer+cite →
correct→propose → certify→supersede). See
[diagrams/architecture-and-roadmap.html](diagrams/architecture-and-roadmap.html)
for the visual version of this file.

Each phase has an **exit gate** — a concrete, checkable condition, not a vibe.
Don't start the next phase's design work until the current gate is met; it's
fine to *plan* ahead, just not to build ahead.

---

## Phase 0 — Walking skeleton ✅ DONE (2026-08-20)

- [x] Docker stack: `api` · `worker` · `migrate` · `postgres+pgvector`
- [x] Versioned knowledge store, supersession, RLS tenancy, attribution logging
- [x] Proposal → certify → materialize loop; impact preview v1 (direct dependents + answer-usage count)
- [x] LLM provider abstraction (`echo` / Anthropic / OpenAI-compatible for exo/ollama)
- [x] Running locally via Docker Desktop

**Exit gate:** `make demo` passes all 14 checks — met, on both the cloud sandbox and local Docker Desktop.

---

## Phase 1 — First real domain + evals  🚧 IN PROGRESS

> Task board with dependencies and the exit-gate checklist: the task board in the private implementation repo

- [x] **Repo on GitHub + CI running the walking skeleton on every push** (2026-10-07)
- [x] **Migration 0002 — entity layer:** entities, aliases, typed object→entity edges, knowledge gaps; edges declared on proposals and materialized at certification; one-hop expansion in compilation; impact preview reports entity scope (2026-10-07, pulled forward from Phase 2 because the HRMS pack needs entities for scope anyway)
- [x] **Migration 0003 — ADR-0014 typed decisions:** `decide()` with heuristic + Jev providers; risk auto-scoring on `/proposals`; grounded-or-refuse gate on `/ask` (2026-10-08; rebuilt after the Oct 5 implementation was lost)
- [x] Domain-pack loader: YAML → entities + proposals (install = diff + certify, zero platform code) (2026-10-08)
- [x] HRMS pack v0: entity types from the EazeWork policy-parameter matrix, `policy_qa` agent, seeds + golden set (2026-10-08). `support_drafting` (UC-H7) still to define per doc 07
- [x] UC-H0 policy-from-configuration prototype (doc 07 §2.1) + batch-certify CLI (2026-10-08, on a synthetic snapshot; real export parser pending)
- [ ] Real LLM routing (task-class → provider table) + first embeddings provider chosen (sticky — ADR-0009)
- [x] Eval harness: golden set 25/26 (1 xfail), no-regression check held across repeats, wired into `make demo` and CI (2026-10-08). Baseline comparison and ticket-mined v2 wait on a real LLM and EazeWork's ticket export
- [ ] Jev access decision: hosted vs. self-hosted — the eval harness arbitrates

**Exit gate:** golden set green **and beats the bare-model baseline**; a deliberately-introduced correction is shown to never regress across repeated runs; grounding and refusal rates reported from `decision_traces`.

## Phase 2 — Graph Stage 2 + capture

- [ ] Impact preview: recursive traversal (depth 2–3) over the 0002 edges, replacing direct-dependents + scope-siblings
- [ ] Relationship ontology in packs (`ontology/relationships.yaml`): valid edge kinds per type pair, validated at proposal time — for HRMS the 21×27 matrix nearly verbatim
- [ ] Conflict detector: SQL recall (scope-edge + valid-time overlap) → `decide()` precision filter → human queue
- [ ] Bootstrap ingestion (UC-H6): documents → candidate objects → `decide()` classifies type + sensitivity → certification batch
- [ ] Risk rubric v1: calibrated `risk_probability` thresholds per domain posture
- [ ] Auto-materialize policy for low-risk + agent-provenance + same-domain, with sampled post-hoc review

**Exit gate:** multi-hop golden questions pass; impact preview traverses beyond direct dependents; at least one proposal auto-materialized by policy and caught by sampled review.

## Phase 3 — Products, reports, second domain

- [ ] Data products (contract-versioned) + `ReportTemplate` schema
- [ ] Scheduler (periodic jobs on the existing Postgres queue) + markdown→HTML/PDF rendering
- [ ] Report delta attribution (data changed vs. definition changed vs. policy changed)
- [ ] Gathr domain pack: Sentinel/test agent, aggressive auto-materialize + sampled review — the platform's staging ground for autonomy policy
- [ ] Review-gated reports (reuse the certification queue)
- [ ] Move to Mac Mini: registry-pulled images, nightly backup + **rehearsed** monthly restore, Tailscale access

**Exit gate:** first recurring report ships itself on schedule; a second tenant runs in production with proven RLS isolation.

## Phase 4 — Advisors & scale

- [ ] Steward + Advisor + Reporter composite agent (the taxation-agent pattern) on a real corpus
- [ ] Layer-2 derived graphs / GraphRAG-style extraction — **only if** Stage 1/2 traversal hits a measured latency wall (Ontology doc §5, Stage 3 tripwire)
- [ ] Campaigns domain pack; interpretation/learning corpus compounding
- [ ] Cloud re-homing: managed Postgres first (per Deployment doc §11 mapping table); the eval golden set doubles as the migration acceptance test

**Exit gate:** a rule/policy change flows end-to-end — blast radius computed → impact memo drafted → dependent interpretations flagged stale → re-certified.

---

## Parking lot (explicitly deferred, not forgotten)

- Graph database (Neo4j/AGE) — only past the Stage 3 tripwire in the Ontology doc
- Admin UI — CLI + `/docs` (FastAPI) carries the operator through Phase 3
- Email/Slack report distribution — stored runs + notification is enough through Phase 3
- Multi-operator auth (OIDC) — single operator API keys are sufficient until someone besides you certifies proposals
