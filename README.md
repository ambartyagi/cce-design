# CCE — Cumulative Context Engine: design papers and decision log

CCE is a compliance engine for HR and employment law, built on Claude and operated by a team of
specialised agents. It answers statutory questions against regulatory corpora (India's four Labour
Codes and the Digital Personal Data Protection Act first; US state leave and final-pay law next) for
a design partner in the Indian HR software market, where a wrong answer about a wage definition or a
retention window is a legal exposure for the client.

This repository holds the **design papers, architecture notes and every architecture decision
record** for CCE. The implementation lives in a private repository; nothing here depends on it.

Author: Ambar Anant. Design period: September to November 2026. Status: pilot cohort of agents
going live with the design partner; the governance rules below are specified and built, not yet
measured in production.

## Start here if you care about AI control

Most of the engineering went into *control* rather than capability. Three documents carry that
story:

1. **[ADR-0028 — Bot autonomy is a per-action-type level, earned by record and lost automatically](adr/0028-autonomy-ladder-per-action-type.md)**
   The autonomy ladder: L0 observe → L1 propose → L2 act behind a calibrated gate with undo → L3 act
   and notify. Only reversible actions may reach L2. Two-week probation, promotion earned by a
   declared record (e.g. 50 consecutive approvals without edit), demotion automatic and immediate on
   the first rollback, client flag, budget breach or approval-rate collapse. Levels are re-earned,
   never re-granted by hand.
2. **[Design 10 — Bot Operating Model](design/10-bot-operating-model.md)**
   Bots as principals with runs, decisions and budgets; six lanes; staged hiring; why bots never
   call each other.
3. **[Architecture 02 — Governance](architecture/02-governance.md)**
   The gates every change passes through, and the rule that code changes and knowledge changes
   never ship together, so an eval regression can always be attributed.

Supporting decisions: [ADR-0013 two-layer knowledge, certified vs derived](adr/0013-two-layer-knowledge-certified-vs-derived.md) ·
[ADR-0014 typed decision models as governance gates](adr/0014-typed-decision-models-for-governance-gates.md) ·
[ADR-0027 ops-instance bots as principals](adr/0027-ops-instance-bots-as-principals.md) ·
[Design 06 — calibrated typed decisions](design/06-jev-typed-decision-layer.md) ·
[Architecture 04 — change paths](architecture/04-change-paths.md) · [Architecture 05 — testing](architecture/05-testing.md).

## What is not yet known

The mechanisms are good enough to ship a pilot. They are not research, and the open questions are
stated in the documents rather than hidden:

- The thresholds (fifty approvals, two weeks) come from control-systems intuition about delays and
  oscillation, not from measurement.
- Approval is a weak signal. A human approving an action is evidence the action *looked* fine. Over a
  long streak of easy questions the ladder cannot tell the difference, and demotion fires only on a
  *reported* failure.
- The ladder is legible to the model. The rules live in the system's own context, and there is no
  test yet for whether a capable model optimises for approval rather than correctness.
- Evals drift toward the deployer's blind spots.

## Read the design in order

| # | Doc | What it answers |
|---|---|---|
| 00 | [Problem Statement & Premise](design/00-problem-statement-and-premise.md) | Why build this at all; the core thesis (context compounds outside model weights); the four-layer architecture |
| 01 | [Design Review](design/01-design-review.md) | Critical gaps in v0.1: compilation, capture loop, evals, temporal semantics, PII/erasure, threat model, lineage, rollback, certification bottleneck, cold start |
| 02 | [Docker Deployment Design](design/02-docker-deployment-design.md) | Service topology, image/build strategy, operations, cloud migration path |
| 03 | [Use Cases & Domain Config](design/03-use-cases-and-domain-config.md) | 19 use cases; the domain-pack configuration model |
| 04 | [Reporting & Agent Archetypes](design/04-reporting-and-agent-archetypes.md) | Reports as governed data products; seven agent archetypes; a worked example |
| 05 | [Ontology & Graph Layer](design/05-ontology-and-graph-layer.md) | Ontologies and knowledge graphs; the certified-vs-derived rule; staged adoption |
| 06 | [Typed Decision Layer](design/06-jev-typed-decision-layer.md) | Calibrated typed decisions (risk, grounded-or-refuse, question kind) as governance gates |
| 07 | [EazeWork Discovery & Design Impact](design/07-eazework-discovery-and-design-impact.md) | Policies live in configuration, not documents; the Policy Parameters matrix as the HRMS ontology |
| 08 | [Widget Config Export Fit](design/08-widget-config-export-fit.md) | How the first real configuration export maps onto the design |
| 09 | [Roles, Sync Plans & Settings](design/09-roles-plans-and-settings.md) | Who is who; sync triggers, plans and metering; three-level settings |
| 10 | [Bot Operating Model](design/10-bot-operating-model.md) | Running the company on a second CCE: bots as principals; the autonomy ladder; six lanes |

Also:

- **[adr/](adr/)** — 28 Architecture Decision Records, one file per decision, never edited after acceptance.
- **[architecture/](architecture/)** — how the system works now: processes and data flow, governance, versioning, change paths, testing, events.
- **[evals/](evals/)** — golden question sets and the no-regression invariant.
- **[ROADMAP.md](ROADMAP.md)** — phases and exit gates.
- **[diagrams/](diagrams/)** — `architecture-and-roadmap.html` and `knowledge-layers.html`; open in a browser.

## Conventions

Every design doc carries a version and a date; versions are bumped, history is not silently edited.
Whether a decision is Proposed or Accepted lives in the ADR log, not in the design prose. Operational
status, deployment runbooks and the task board stay in the private repository.
