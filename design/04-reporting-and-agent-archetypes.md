# Cumulative Context Engine – Reporting & Agent Archetypes

**Version:** 0.1
**Date:** 2026-08-03
**Companion docs:** Design Document v0.1 · Design Review v0.1 · Docker Deployment Design v0.1 · Use Cases & Domain Config v0.1
**Question addressed:** Can the platform serve as a reporting system (standard + recurring reports), and can it spawn different kinds of agents — e.g., a taxation agent that creates reports and opines on questions and regulatory changes?

---

## 1. Short Answer

Yes — and it's less of a stretch than it might appear. A report is a **data product whose consumers are humans**: a versioned, materialized, lineage-tracked artifact derived from other products and knowledge objects. The platform already owns versioning, materialization, dependency edges, scheduling-capable workers, provenance, and a certification pipeline — which is roughly 70% of a reporting system, and precisely the 70% that ordinary BI tools are weakest at. What's missing is the last 30%: report templates, a scheduler surface, rendering, and distribution.

The strategic point, though, is not that the platform *can* produce reports — anything can. It's that reports produced here inherit properties BI tools don't have:

1. **Semantic grounding.** Every figure is computed under a *certified, versioned metric definition*. "Attrition rate" in the March report and the April report either means the same thing, or the report says explicitly that the definition changed between runs — and links the supersession.
2. **Self-explaining change.** Because reports sit in the dependency graph, "why is this number different from last month?" has a computable answer with three possible causes: the data changed, a definition changed, or a policy changed. The report can attribute its own deltas. This is the reporting feature people actually want and almost never get.
3. **Narrative with citations.** The prose sections cite knowledge objects the way answers do (attribution machinery, unchanged). An auditor can trace a sentence to a certified rule.
4. **Reports as capture surfaces.** A user correcting a report ("that's not how we count contractors") is correcting the *underlying definition* — the correction flows through the normal proposal pipeline and fixes every future report and answer, not just this document. A BI tool correction fixes one dashboard.

So: yes, look at it as a reporting system — specifically as *the reporting system for organizations that want reports to be accountable to definitions*.

---

## 2. Reports as Data Products

### 2.1 The model

A **ReportTemplate** is a pack artifact (like product templates); a **ReportRun** is a materialization of it:

```
ReportTemplate (in domain pack, versioned)
  ├─ structure: sections, each = computed table/figure slot | narrative slot
  ├─ inputs: data products + knowledge-object queries (declared → dependency edges)
  ├─ metric bindings: which MetricDefinition versions the computations use
  ├─ schedule: cron | event-trigger | on-demand
  ├─ audience + sensitivity class
  ├─ distribution: where runs go (v1: stored + notification; later: email/Slack)
  └─ review gate: none | sampled | required   ← reuses the certification pipeline

ReportRun (materialized by worker)
  ├─ immutable, versioned, tenant-scoped artifact (markdown → rendered HTML/PDF)
  ├─ full attribution: object versions + product versions + model/prompt versions used
  ├─ delta section: diff vs. previous run, attributed to data/definition/policy changes
  └─ status: draft → (reviewed →) published
```

### 2.2 The one hard rule: numbers from queries, words from models

The LLM never computes a figure. Numeric sections are produced by the product layer (SQL against materialized products, under pinned definition versions); the agent writes *around* computed results — narrative, interpretation, anomaly commentary. This single rule is what makes the reports trustworthy and is non-negotiable in every archetype below. It also means report correctness decomposes cleanly: numeric bugs are product/definition bugs (testable in CI), narrative bugs are grounding bugs (caught by the eval harness and review gates).

### 2.3 Report kinds, mapped to existing machinery

| Kind | Trigger | Machinery | Example |
|---|---|---|---|
| Standard | On demand | ReportTemplate + worker job | Leave-liability statement for an entity |
| Recurring | Schedule | Same + periodic jobs (the Postgres queue already supports cron-style periodic tasks — no new service) | Monthly TDS summary; weekly Gathr stability report |
| Triggered | Domain event | Same + event hook from connectors/agents | Campaign post-mortem report when a campaign closes; incident report when monitoring agent closes an incident |
| Regulatory-response | Knowledge change | Impact analysis on a certified change → report drafted from the blast radius | "What the new Finance Act changes for us" (see §5) |

The fourth kind is the novel one: **the impact-analysis output is itself the seed of a report.** No BI tool has this because no BI tool knows what a rule change *touches*.

### 2.4 What genuinely has to be built (the honest 30%)

- **ReportTemplate schema + pack support** — small; it's a sibling of product templates.
- **Scheduler surface** — periodic-job registration from pack install; a `report_runs` table; catch-up semantics for a Mini that was asleep at 2am (run-once-late, never double-run).
- **Rendering** — markdown-first (agents are good at it), rendered to HTML always and PDF when a template demands it (WeasyPrint in the worker image; ~50MB, acceptable). Charts, when needed, are generated from computed data by deterministic code, not by the model.
- **Review gate wiring** — a ReportRun with `review: required` enters the same certification queue as proposals. HIGH-sensitivity reports (anything statutory) are drafts until a human publishes. This is reuse, not new machinery, and it's the governance answer to "can an agent send a tax report to a client?" — it can *draft* one.
- **Distribution v1** — stored runs + notification. Email/Slack connectors are a later, separable step.

Deployment impact: zero new services. Worker gains periodic jobs and a rendering dependency; storage grows by rendered artifacts (Postgres large objects at POC scale, object storage later per the deployment doc's noted path).

---

## 3. Agent Archetypes

"Spawn different kinds of agents" is already half-true in the domain-pack design — agent definitions are pack artifacts. What the taxation example adds is the recognition that agents cluster into **archetypes**: platform-level patterns of (tools + compilation policy + proposal rights + risk posture), which packs instantiate with domain configuration. Formalizing them keeps the platform honest (new archetype = platform work; new instance = configuration).

| Archetype | Does | Proposal rights | Existing examples |
|---|---|---|---|
| **Oracle** | Answers questions, grounded, cites | May propose from corrections | policy_qa (H1/H2), rules oracle (G3) |
| **Analyst** | Explains data/anomalies against definitions | Definition-gap proposals | payroll explainer (H3), balance analysis (G5) |
| **Reporter** | Materializes ReportTemplates, writes narrative, attributes deltas | Report drafts; definition-gap proposals | *new — enabled by §2* |
| **Sentinel** | Watches streams, matches patterns, escalates | Pattern/runbook proposals, auto-materialize per posture | monitoring agent (G2) |
| **Elicitor** | Structured interviews → batched knowledge proposals | Bulk proposals | post-mortem (C3), playtest (G4) |
| **Steward** | Watches an *external corpus* (regulations, platform ToS, game meta) for changes; ingests, diffs, proposes updates + impact memo | Change proposals + regulatory-response reports | *new — the taxation agent's engine (§5)* |
| **Advisor** | Opines: applies certified interpretations to a situation and drafts a position | Interpretation proposals (always human-certified) | taxation agent's front end (§5) |

Two are new to the design: **Reporter** (needs §2's machinery) and **Steward** (needs a corpus-watch connector pattern: fetch → diff → extract candidate changes → propose). **Advisor** is an Oracle variant with stricter rules — see the risk framing in §4. Composite agents are configurations that chain archetypes (the taxation agent = Steward + Advisor + Reporter sharing one knowledge base), not new code.

---

## 4. Opining Safely: the Fact / Interpretation / Advice Ladder

An agent that "opines" needs a sharper knowledge taxonomy than one that answers. Three rungs, with escalating governance:

1. **Fact about a rule** — "§80C limit is ₹1.5L." Provenance: statutory source. Certification: standard. The agent states these freely when grounded.
2. **Interpretation** — "We treat internet reimbursement as exempt under X, based on Y." A *choice among defensible readings*, made by a qualified human, stored as a certified `Interpretation` object with author, rationale, and the rule versions it depends on. **Agents apply interpretations; they never mint them** — an agent may draft a candidate interpretation, but it exists only as a proposal until a human expert certifies it. This rung is where the compounding is most valuable: the org's accumulated positions, versioned and reusable.
3. **Advice to a person/entity** — "You should choose the new regime." Highest risk. The agent's ceiling here is: compute the comparison under certified definitions (numbers from queries), present it with the interpretations applied and cited, flag where certified knowledge runs out, and carry the disclaimer that it is not professional tax advice. `grounded_or_refuse: true` is mandatory; silence beats plausible improvisation.

When a rule changes and interpretations sitting on it are invalidated — the dependency graph catches exactly this (interpretations declare edges to rule versions) — affected interpretations are auto-flagged `stale`, the humans who own them are queued, and agents stop applying them until re-certified. **This is the review's impact-analysis machinery doing its most valuable single job.**

---

## 5. Worked Example: the Taxation Agent

Configuration, not platform code: a composite agent in the HRMS pack (taxation is a subdomain — its shared statutory layer is the same mechanism as compliance knowledge, country-scoped).

**Knowledge types added to the pack:** `TaxRule` (effective-dated, shared, country-scoped — note both fiscal-year valid-time and enactment-vs-effective dates, the bitemporal case from the Review made vivid), `Interpretation` (tenant- or platform-scoped, human-certified only), `TaxPrecedent`, `FilingRequirement`.

**Its four jobs, mapped to archetypes:**

1. **Steward — watch the rules.** A corpus connector ingests statutory updates (budget/Finance Act changes, circulars — v1 can be "operator uploads the change document"; automation later). The agent diffs against current `TaxRule` objects, proposes supersessions with effective dates, and the impact preview computes the blast radius: dependent interpretations, report templates, and — via the employee-context product — *how many employees each change touches*.
2. **Advisor — opine on the change.** From that blast radius it drafts the impact memo: "New §87A rebate threshold affects ~40% of employees in tenant X; interpretations I-12 and I-17 need review; December TDS projections shift." Numbers from queries, framing from certified interpretations, gaps flagged. The memo is a `review: required` ReportRun — a human expert publishes it.
3. **Reporter — recurring outputs.** Monthly TDS summary per tenant/entity; quarterly filing-prep checklist against `FilingRequirement` objects; year-end proof-collection status. All standard §2 machinery.
4. **Oracle/Advisor — answer questions.** Employee regime comparisons (computed, cited, disclaimed), HR-admin queries on treatment of a component ("as per certified interpretation I-12, certified by R. Mehta, dependent on TaxRule §17(2) v3").

**What compounds:** the interpretation corpus — the org's accumulated tax positions — plus precedents and the memo history. Year two's budget-change response takes hours instead of weeks because last year's blast-radius analysis, positions, and memo structure are all in the store. That sentence is the platform pitch in miniature, and it generalizes: swap the corpus and you have a **labor-law agent**, a **platform-ToS agent** for Campaigns (Instagram policy changes → which creator playbooks are affected), or a **meta-watch agent** for Gathr. The Steward+Advisor+Reporter composite is a *franchise*, and taxation is its first instance.

---

## 6. Fit Check and Roadmap Placement

**Boundary honesty:** this does not make the platform a BI tool — no slice-and-dice exploration, no dashboard builder, no sub-second OLAP. Interactive analytics stay in BI tools reading the *same governed data products* (products can expose views BI connects to — the semantics stay singular even when the pixels are elsewhere). The platform's reporting lane is **narrative, governed, self-explaining documents** — the things BI is bad at.

**Sequencing.** Reporting lands after the walking skeleton and first Oracle use case, not before — it depends on products, definitions, and attribution being real. Recommended insertion into the implementation roadmap:

1. *(existing)* Walking skeleton → HRMS `policy_qa` live.
2. **Reporter v1:** ReportTemplate + scheduler + markdown/HTML rendering + delta attribution; first recurring report in whichever domain is live (leave-liability monthly is a good first: numeric, boring, easy to verify).
3. **Review-gated reports:** wire ReportRuns into the certification queue; PDF rendering.
4. **Steward v1** (manual-upload corpus watch) + `Interpretation` type + staleness flagging — this is the taxation agent's minimum viable core.
5. **Taxation composite** assembled from the above; impact memos as regulatory-response reports.
6. Distribution connectors (email/Slack) when someone other than the operator consumes reports.

Steps 2–4 are each small because they lean on existing machinery; the genuinely new surfaces are the template schema, the scheduler, rendering, and the corpus-diff connector pattern.

**New open questions for the running list:** report retention/archival policy per sensitivity class (statutory reports may have mandated retention — the mirror image of erasure); whether ReportRuns embed snapshot data or re-derive on view (recommend: embed — reports are historical documents and must not silently change); how far chart generation goes before it's scope creep (recommendation: computed tables + a handful of deterministic chart types, nothing more).

---

*Companion to the CCE design series. If accepted, fold §3's archetype table into the core design doc and §5's knowledge types into the HRMS domain pack.*
