# 0027. A second CCE instance runs the company; bots are principals with runs and decisions

Date: 2026-10-16
Status: **Proposed** (design: [design/10-bot-operating-model.md](../design/10-bot-operating-model.md); amended 2026-10-17, see below)

## Context

CCE will be operated by one person for roughly six months and by a few people after that. Most
operational work — syncs, onboarding, statute watching, evals, reporting, pipeline — is to be done
by bots. Bots need job descriptions that change by review, a record of every unit of work with
undo, a place for a human's rulings, an inbox, and identities with scoped access. CCE already has
each of these for knowledge (ADR-0013, ADR-0022, ADR-0017, ADR-0024, ADR-0023). Repo bots and
founder-side bots must never hold production tenant credentials.

## Decision

1. **A second CCE instance — the Ops instance — runs the company.** Same image, separate database,
   an `ops` domain pack of four knowledge types (`BotSpec`, `OperatingRule`, `Milestone`, `Lane`)
   and one report template (the daily brief). *Tenant layout amended 2026-10-17 — see below.*
2. **A bot is a service principal** (ADR-0024), one per tenant it works in, with a role made of
   existing permissions. Everything it does is recorded against that principal.
3. **A bot spec is a knowledge object**, sourced from `bots/<name>/*.yaml` in the repo, installed
   by the pack loader, superseded on change. A spec must declare what it reads and writes, each
   action type with its reversibility and starting autonomy level, budgets, an escalation lane
   and an eval set.
4. **A bot run is a sync run generalised**: bot, spec version, build SHA, trigger, input
   fingerprint, outputs, cost, outcome, `decided_by`, status `staged | held | applied |
   discarded | rolled_back`. Rollback reverts every change the run made.
5. **A decision is a Clarification** in the owning tenant, opened as a question for the founder
   when a run is held, pinned to the `BotSpec` action version it was made under (ADR-0022 §5).
   Consistent decisions are generalised by proposal into `OperatingRule`s that a human certifies.
6. **Bots do not message people and do not call each other.** They write runs, proposals and
   questions, and react to outbox events (ADR-0023). One Reporter bot produces one brief a day.
7. **The Ops instance reads production only through production's outbox**, via a connector, like
   any external corpus. Production has no knowledge of the Ops instance. No client content enters
   the Ops instance — ids and counts only (design review G6).

## Amendment (2026-10-17): platform/client tenants instead of one tenant

The original decision said "one tenant (`navoiq`)". Revised while standing up the instance:
the Ops instance reuses the ADR-0015 structure one level up —

- **`Navoiq` is the platform tenant**: cross-app OperatingRules, Milestones, Lanes, autonomy
  settings, founder decisions that span apps.
- **Each governed app is a client tenant** (`CCE`, `Gathr`): its bot runs, runbooks, eval
  knowledge and app-scoped decisions, inheriting the platform layer.

Two reasons: it keeps each app's ops isolated under RLS exactly as client companies are in
production, and it pre-proves the multi-tenant shape "CCE Ops" needs if the Day-90 LOI test
turns it into a product — a pilot customer is then simply one more client tenant. Constraint
unchanged: a `Gathr` tenant may receive events (CI, evals), but Gathr remains parked (STRATEGY
§5) — no Gathr build items may originate from it.

## Alternatives considered

- **A `navoiq` tenant inside production.** Rejected: repo and founder-side bots would need
  production keys, and a company-operations tenant beside client tenants widens the blast radius
  of any RLS mistake.
- **One flat tenant for all of Navoiq** (the original form). Superseded by the amendment: it
  mixed every app's runs and rules in one namespace and proved nothing about multi-tenancy.
- **A separate bot-orchestration framework.** Rejected: it would rebuild versioning, review,
  undo, identity and events that CCE already has, and bots would stop being independently
  debuggable and switch-off-able.
- **Bots messaging the founder directly** (Slack, email per event). Rejected: the inbox would
  scale with bots × companies; a single ranked brief scales with what matters.

## Consequences

- One migration in the Ops instance's pack (bot runs as a `kind` on `source_snapshots` or a
  sibling table; `BotSpec` and `OperatingRule` types); the `cce_events` connector; a `github`
  connector for repo bots to record runs.
- Dogfooding: CCE becomes its own first customer in a non-commercial second domain, which tests
  "domain-agnostic" without violating STRATEGY §5 — now including multi-tenancy of the ops pack.
- A second database to back up. Its loss costs decisions only; specs are in git and events in
  production's outbox.
- Deployment: same host as the pilot is acceptable (separate compose project, separate volume,
  localhost-only port); see `deploy/ops/OPS-SETUP.md`.
