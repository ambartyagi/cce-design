# Cumulative Context Engine – Use Cases & Domain Configuration

**Version:** 0.1
**Date:** 2026-08-03
**Companion docs:** Design Document v0.1 · Design Review v0.1 · Docker Deployment Design v0.1
**Purpose:** Ground the platform in concrete use cases for each target domain (HRMS, Influencer/Campaigns, Gathr), and define how the platform is configured per domain.

---

## 1. How to Read the Use Cases

A use case only belongs on this platform if it passes the **compounding test**: it must both *consume* institutional context and *produce* new context as a side effect of being used. A use case that only consumes is a plain RAG app; one that only produces is a data-entry tool. The compounding loop is the product.

Each use case below is therefore described on both sides of that loop, plus its risk class (which drives the certification rubric from the Review, §2/G9):

- **Consumes:** knowledge types and data products the agent needs at compile time
- **Produces:** what flows back into the store — corrections, precedents, learnings, new objects
- **Risk:** LOW (auto-materialize + sampled review) / MED (batch human certify) / HIGH (blocking human certify)
- **Tier:** 1 = first wave (read-only, assistive) · 2 = second wave (workflow-embedded) · 3 = later

The tiering is deliberate: **Tier 1 use cases answer questions and never write to the source system.** They exercise compilation, tenancy, provenance, and the correction loop with near-zero blast radius — exactly the walking-skeleton philosophy extended into real domains.

---

## 2. HRMS Domain (EazeWork-style, multi-tenant, multi-country)

### 2.1 The domain in one paragraph

Hundreds of client organizations (tenants) run HR, payroll, leave, attendance, and performance on a shared platform. Truth is layered: statutory rules per country (shared across all tenants operating there), platform default configurations, tenant-specific policies, and per-employee exceptions. Everything is effective-dated, much of it is auditable by law, and the people asking questions range from employees ("how much leave do I have?") to HR admins to EazeWork's own implementation teams. This is the domain that stresses **tenancy, temporal semantics, and compliance** hardest — which is why it validates the platform's core claims.

### 2.2 Use cases

**UC-H1 · Employee policy Q&A** — Tier 1 · Risk LOW
An employee asks about leave, payroll components, reimbursement rules, notice periods. Agent answers *for their tenant, grade, location, and date*, citing the policy objects used.
Consumes: PolicyRule (effective-dated, tenant-scoped), Definition (e.g., what counts in "CTC"), ComplianceRequirement (statutory floor/ceiling), employee-context data product (grade, location, tenure — read-only).
Produces: unanswerable/ambiguous questions logged as knowledge-gap signals; HR's clarifications become PolicyRule proposals. Repeated questions on the same topic are the passive signal (Review §3) that a policy object is missing or unclear.
*This is the recommended first real use case — it is the HRMS slice from the Review, made concrete.*

**UC-H2 · HR-admin policy assistant with "why"** — Tier 1 · Risk LOW
HR admins ask harder versions: "What's our notice period for managers in the UAE entity, and when did it change?" The differentiator vs. UC-H1 is **provenance display**: answers show which object versions, who certified them, effective dates, and supersession history. This is where the context store visibly beats a chatbot.
Consumes: same as H1 plus full version history. Produces: corrections from the people most qualified to make them — the highest-quality capture channel in the whole domain.

**UC-H3 · Payroll anomaly explanation** — Tier 1 · Risk MED
"Why did this employee's PF deduction change this month?" The agent reconstructs the explanation from effective-dated rules, definitions, and the payroll-register data product. Read-only, but MED risk because a wrong explanation about money erodes trust disproportionately — answers must be grounded-or-refuse (no answer without cited objects).
Consumes: MetricDefinition (each payroll component), PolicyRule, ComplianceRequirement, payroll-register product. Produces: gaps in metric definitions; precedent explanations worth certifying as reusable objects.

**UC-H4 · Compliance calendar Q&A** — Tier 1 · Risk LOW
Statutory filings, deadlines, thresholds per country/entity. Notable because it runs almost entirely on **shared platform knowledge** (India PF rules are true for every tenant operating in India) — the use case that justifies the shared-knowledge layer and exercises tenant inheritance/pinning semantics.

**UC-H5 · Implementation/configuration assistant** — Tier 2 · Risk MED
EazeWork's implementation teams configure modules for new clients. The agent suggests configurations based on similar past implementations and — critically — **captures the decisions and their rationale as tenant knowledge** ("client chose 26-day payroll basis because…"). Implementation decisions are today's most-lost tribal knowledge; this use case is a capture machine disguised as an assistant.
Consumes: ProcessPlaybook, ImplementationPrecedent (cross-tenant, anonymized-pattern level — see §5.4 policy), platform defaults. Produces: TenantOverride objects, new precedents, playbook refinements.

**UC-H6 · Tenant policy bootstrap** — Tier 2 · Risk MED
New tenant onboarding: ingest their policy documents, extract candidate PolicyRule objects, diff against platform defaults, queue the batch for certification. This *is* the cold-start plan (Review G10) expressed as a product feature — the first stress test of batch certification UX.

**UC-H7 · Employee-query response drafting for HR** — Tier 2 · Risk MED
Agent drafts responses to employee tickets grounded in certified policy; HR edits and sends. Every edit is a correction signal on the underlying objects.

**UC-H8 · Payroll-run pre-check** — Tier 3 · Risk HIGH
Before payroll finalization, an agent reviews the run against known rules and past anomaly patterns and flags outliers. High stakes, write-adjacent, needs a mature store — a destination, not a start.

### 2.3 HRMS knowledge types and data products

Object types (ontology extension): `PolicyRule` (effective-dated, scoped tenant/entity/country/grade), `Definition`, `ComplianceRequirement` (shared, country-scoped), `TenantOverride`, `ApprovalPrecedent`, `ImplementationPrecedent`, `ProcessPlaybook`.
Data products: `employee-context` (read-only slice: grade, location, tenure — heavy PII discipline), `policy-catalog`, `payroll-register` (read-only, HIGH sensitivity), `compliance-calendar`.

### 2.4 What compounds here

Statutory interpretations, tenant policy clarifications, exception precedents, implementation rationale. Note the leverage: a certified interpretation of an ambiguous statutory rule benefits *every tenant in that country* — shared knowledge is where HRMS context compounds super-linearly.

---

## 3. Influencer / Campaigns Domain

### 3.1 The domain in one paragraph

Campaign performance lives or dies on definitions that are tribal and drifting ("engagement rate" means different things per platform and per client), on learnings that evaporate after each campaign post-mortem, and on relationship memory (which creators deliver, at what rates, for which audiences). Tenants map to **brands/clients**; much of the interesting knowledge (creator track records, rate benchmarks) is arguably agency-level shared knowledge — the inverse emphasis from HRMS, where private-per-tenant dominates. This domain stresses **semantic definitions and analytics-flavored data products**.

### 3.2 Use cases

**UC-C1 · Grounded performance Q&A** — Tier 1 · Risk LOW
"Which creators drove the best ROI for beauty campaigns last two quarters?" — answered *using certified metric definitions*, with the definition version cited. The differentiator: when the definition of ROI changes, old answers are explainable and new answers are consistent. Consumes: `MetricDefinition`, `AudienceSegmentDefinition`, campaign-performance product. Produces: definition disputes → correction proposals (the most common correction type in this domain).

**UC-C2 · Creator recommendation with memory** — Tier 1 · Risk LOW
Shortlist creators for a brief using certified learnings: past delivery vs. commitments, audience overlap issues, content-style fit. Consumes: `CreatorLearning`, `RateBenchmark`, `BrandGuideline`, creator-roster product. Produces: post-selection outcomes feed back as new learnings (was the recommendation right?) — a rare use case with a *measurable* feedback loop, which makes it a strong eval target.

**UC-C3 · Post-mortem capture** — Tier 2 · Risk LOW–MED
After each campaign, the agent interviews the campaign manager (structured elicitation: what worked, what didn't, why, against which metrics), drafts `CampaignLearning` objects, and proposes them. This is the domain's **primary capture engine** — worth building early even though it "answers" nothing, because C1/C2 quality depends on the supply it creates.

**UC-C4 · Brief and proposal drafting** — Tier 2 · Risk MED
Draft campaign briefs from brand guidelines + relevant past learnings + audience definitions. Human always edits; edits are correction signals.

**UC-C5 · Rate/negotiation memory** — Tier 2 · Risk MED
"What did we pay this creator tier last time, and what's a fair band?" Sensitive (commercially confidential per client — tenancy question §3.3), high value, structurally simple.

**UC-C6 · Audience definition management** — Tier 1 · Risk LOW
"Gen-Z tier-2 India" as a *versioned, certified object* with explicit criteria rather than a phrase that means something different in every deck. Small, unglamorous, and the fix for the domain's most chronic silent error.

### 3.3 Domain-specific design note: the shared/private split is commercial

Creator track records and rate benchmarks compound across clients (agency-level shared knowledge), but clients may reasonably expect their campaign learnings to be private. The default from the core design (private unless explicitly shared) is right, with a per-object-type sharing policy in the domain pack: `CreatorLearning` → shared by default, `CampaignLearning` → private, `RateBenchmark` → shared but banded. This is a *configuration*, not new machinery.

### 3.4 What compounds here

Definitions stabilize, creator memory deepens, post-mortems accumulate into playbooks ("micro-influencer bundles beat single-macro for tier-2 launches"). Compounding is fastest-visible here of the three domains: after ~10 campaigns the with-store vs. without-store recommendation quality gap should be demonstrable on the eval set.

---

## 4. Gathr Domain (card game)

### 4.1 The domain in one paragraph

A live game accumulates exactly the kind of knowledge that evaporates: edge-case rulings, bug patterns, incident signatures, balance-change outcomes, playtest insights. Single tenant, no compliance surface, and the primary knowledge *producers and consumers are agents themselves* — making Gathr the ideal proving ground for **agent-authored knowledge and aggressive auto-materialization**, the platform's most autonomous configuration.

### 4.2 Use cases

**UC-G1 · Test agent with institutional memory** — Tier 1 · Risk LOW
Generates and runs test scenarios informed by `GameRule`, `EdgeCaseRuling`, and — crucially — `BugPattern` objects from every previously found bug. **Every new bug found becomes both a knowledge object and a golden regression case**: this use case *is* the no-regression invariant (Review §6.1) operating as a product. Produces: new BugPatterns and TestScenarios, auto-materialized, sampled weekly.

**UC-G2 · Monitoring agent with incident memory** — Tier 1 · Risk LOW–MED
Watches logs/telemetry; matches anomalies against `IncidentSignature` objects ("this error-spike shape = matchmaking deadlock, check X first"); proposes new signatures and `RunbookEntry` objects after each novel incident. Consumes the telemetry data product; produces the on-call knowledge nobody ever writes down. MED only for runbook *actions*; pattern-matching alerts are LOW.

**UC-G3 · Rules oracle for developers and players** — Tier 1 · Risk LOW
"What happens if two players trigger X simultaneously?" Answered from certified rules and rulings; unanswered questions become `EdgeCaseRuling` proposals for the designer — turning every ambiguity someone hits into a permanent ruling.

**UC-G4 · Playtest feedback capture** — Tier 2 · Risk LOW
Structured elicitation after playtests (mirror of UC-C3), producing candidate balance observations and rule-clarity issues.

**UC-G5 · Balance analysis** — Tier 3 · Risk MED
Win-rate and meta analysis against certified metric definitions, with `BalanceChangeOutcome` objects recording what past nerfs/buffs actually did — institutional memory for the game's hardest recurring decision.

### 4.3 What compounds here

The bug-and-incident corpus, the ruling corpus, and balance history. Gathr also compounds something meta: because risk tolerance is high, it is where **agent-proposed → auto-materialized → sampled-review** workflows get tuned before HRMS ever trusts them. Gathr is the platform's staging environment for autonomy policies.

---

## 5. Cross-Domain View

### 5.1 What each domain proves

| Dimension | HRMS | Campaigns | Gathr |
|---|---|---|---|
| Tenancy | **Hard multi-tenant, RLS-critical** | Multi-brand, commercial sharing policy | Single tenant |
| Temporal (valid-time) | **Critical** (effective-dated policy, payroll) | Mild (definition versions) | Mild (rule versions) |
| Shared-vs-private knowledge | Shared statutory layer, private policies | **Inverse: shared learnings, private campaigns** | N/A |
| Risk posture | Conservative; HIGH gates common | Moderate | **Aggressive auto-materialization** |
| Primary capture channel | HR-admin corrections, implementation decisions | Post-mortem elicitation | **Agent-authored (bugs, incidents)** |
| Compliance/PII | **DPDP/GDPR, payroll data** | Commercial confidentiality | None |
| Compounding visibility | Slow, high-value | **Fastest demonstrable** | Continuous, automatic |

Together the three cover the platform's full claim surface with almost no overlap — a good validation portfolio. Sequencing recommendation stands from the Review: **HRMS UC-H1/H2 first** (validates the differentiating claims), **Gathr UC-G1 second** (tunes autonomy cheaply, and its bug-corpus loop is the best demo of compounding), **Campaigns third** (benefits most from a mature capture UX). If the priority flips toward speed-of-demo over claim-validation, swap the first two — the platform design is unaffected by the order.

### 5.2 New platform requirements surfaced by this analysis

Writing the use cases exposed requirements not yet explicit in the earlier docs — this is the list to fold into the schema deep-dive:

1. **Grounded-or-refuse mode** per use case (UC-H3): a compilation policy flag — if no certified object covers the question, the agent must say so rather than answer from priors.
2. **Structured elicitation flows** (UC-C3, UC-G4): capture is not only reactive correction; the agent runtime needs an "interview" pattern producing batched proposals.
3. **Per-object-type sharing defaults** (§3.3): the domain pack must configure shared/private/banded per type, not per domain.
4. **Knowledge-gap signals as first-class records** (UC-H1, UC-G3): unanswerable questions are logged, aggregated, and surfaced — the demand side of the capture loop.
5. **Outcome feedback edges** (UC-C2): some knowledge objects have measurable outcomes; the schema should allow linking an object to later evidence for/against it (feeds confidence scores, Review §3).
6. **Cross-tenant pattern extraction policy** (UC-H5): implementation precedents shared across tenants must be pattern-level and anonymized, and per the Review's G6 stance this is never automatic — an explicit certified operation with its own proposal type.

---

## 6. Domain Configuration Model: Domain Packs

### 6.1 The concept

A **domain pack** is a versioned, declarative bundle that instantiates the domain-agnostic platform for one domain. It is a directory in git, installed via CLI into the running system, and every install/upgrade flows through the same governance pipeline as any other change (a pack upgrade is a proposal with an impact preview — packs get no side door).

```
domains/hrms/
├── domain.yaml           # identity, tenancy mapping, defaults
├── ontology/             # object-type definitions (JSON Schema per type)
│   ├── policy_rule.json
│   └── ...
├── agents/               # agent definitions: system prompts, tool grants, compilation policy
│   ├── policy_qa.yaml    #   one per use case, roughly
│   └── ...
├── risk_policy.yaml      # risk rubric bindings for this domain
├── sharing_policy.yaml   # per-object-type shared/private defaults
├── products/             # data product templates (contract, schema, refresh)
├── connectors/           # source configs (API endpoints, ingestion mappings)
├── seeds/                # bootstrap knowledge (certified-on-install, provenance=seed)
└── evals/                # golden question set for this domain
```

### 6.2 What the platform owns vs. what the pack owns

| Platform core (never in a pack) | Domain pack |
|---|---|
| Versioning, supersession, change log | Object types + their JSON Schemas |
| Tenancy enforcement (RLS), auth | Tenancy *mapping* (what a tenant means here) |
| Proposal→certify→materialize pipeline | Risk rubric *bindings* (which changes are HIGH here) |
| Compilation engine, precedence machinery | Compilation *policy* (weights, grounded-or-refuse flags) |
| Provider abstraction, routing table | Routing *preferences* (which task classes may use local models) |
| Eval harness runner | Golden sets |
| Dependency graph, impact analysis | Product templates + their declared dependencies |

The discipline this enforces: adding a domain must require **zero platform code changes**. The first time a domain needs a core change, that's a signal the abstraction is wrong — fix the core, not the pack.

### 6.3 Concrete sketch: `domains/hrms/domain.yaml`

```yaml
domain: hrms
version: 0.1.0
display_name: "HRMS (EazeWork)"

tenancy:
  tenant_means: "client organization"
  shared_knowledge: true          # statutory layer exists
  tenant_inheritance: pinned      # tenants pin shared-object versions;
                                  # upgrades arrive as per-tenant proposals

temporal:
  valid_time_required: [PolicyRule, ComplianceRequirement, TenantOverride]

defaults:
  compilation:
    precedence: [tenant_override, tenant, shared_country, shared_platform]
    grounded_or_refuse: false     # overridden per-agent (payroll agent: true)
  sensitivity_floor: internal     # nothing in this domain is 'public'
```

And one agent definition, `agents/policy_qa.yaml`:

```yaml
agent: policy_qa                  # implements UC-H1
task_class: reasoning             # routing table decides provider (hosted vs exo/local)
tools: [search_knowledge, read_product:employee-context, submit_proposal]
compilation:
  include_types: [PolicyRule, Definition, ComplianceRequirement, TenantOverride]
  as_of: request_date             # valid-time aware
  attribution: required
proposals:
  may_propose: [PolicyRule]       # from user corrections
  auto_materialize: never         # HRMS posture: everything is certified
capture:
  log_knowledge_gaps: true
```

The equivalent Gathr test-agent file differs in exactly the ways §5.1 predicts: `auto_materialize: low_risk`, `sampled_review: weekly`, no valid-time requirements, single-tenant. Same machinery, opposite posture — that contrast is the proof the configuration model is right.

### 6.4 Where configuration lives at runtime

Per the deployment design's rule (env = deployment facts, DB = product facts): packs are **source-of-truth in git, materialized into the database on install** (`cce domain install domains/hrms --env prod`). The install CLI diffs the pack against the DB state and generates proposals for the delta — meaning domain upgrades get impact previews for free. Nothing reads YAML at request time.

### 6.5 Configuring a new domain, end to end

1. Copy the pack skeleton; write `domain.yaml` (tenancy meaning, temporal needs, posture).
2. Define 3–7 object types with JSON Schemas — start minimal; the ontology governance rule covers evolution.
3. Write one agent definition for the first Tier-1 use case.
4. Set the risk and sharing policies (posture table in §5.1 is the guide).
5. Seed knowledge + write 20–30 golden questions *before* going live (the eval set is part of the definition of done for a domain, not an afterthought).
6. Install to dev → walking-skeleton loop against the new domain → install to Mini.

Target: steps 1–6 in under a week for a new domain, most of it spent on seeds and golden questions — i.e., on knowledge, not plumbing. That ratio is the portability claim, measured.

---

## 7. Note on Local LLM via exo

The three-Mac exo cluster fits the existing provider abstraction without design changes: exo exposes an OpenAI-compatible endpoint, so it becomes one more row in the `model_routing` table (e.g., `task_class: extraction → exo`, `reasoning → hosted`). Two disciplines make the experiment cheap and honest: route **per task class** (embeddings and structured extraction are the low-risk places to go local first; agent reasoning last), and let the **golden sets arbitrate** — run the same eval suite against hosted and exo-served models and let the delta, not vibes, decide what routes where. The deployment doc's caveat stands: the cluster is a lab configuration (three consumer machines, availability tied to whether your MacBook is home), so design it as a routing preference with hosted fallback, never a hard dependency.

---

## 8. Next Steps

1. Fold §5.2's six new requirements into the schema deep-dive (they mostly add fields/flags, not tables).
2. Pick the first two agent definitions to fully specify (recommended: `hrms/policy_qa` and `gathr/test_agent` — the posture extremes).
3. Draft the HRMS golden set's first 20 questions from real EazeWork policy questions — the highest-leverage hour of the whole project.
4. Then return to the deferred scaffold: repo + compose + walking skeleton, now with a real domain pack to install into it.

---

*Companion to Design Document v0.1, Design Review v0.1, and Docker Deployment Design v0.1.*
