# 0012. Domains are packs (configuration), never platform code changes

Date: 2026-08-10
Status: Accepted (Use Cases & Domain Config doc §6); domain-pack loader not yet built (Phase 1)

## Context
The platform must support HRMS, Campaigns, and Gathr — and future domains —
without becoming three codebases wearing a trenchcoat. The discipline needs
to be enforceable, not just aspirational.

## Decision
A **domain pack** is a versioned directory (ontology JSON Schemas, agent
definitions, risk/sharing policy, product templates, connectors, seeds,
golden questions) installed via CLI. Install diffs the pack against DB state
and generates proposals for the delta — pack upgrades get impact previews
for free, and get certified like any other change. Platform code owns
versioning, RLS, the proposal pipeline, compilation, and the eval runner;
packs own everything domain-specific. **The first time a domain needs a
platform code change, that's a signal the abstraction is wrong** — fix the
core, not the pack.

## Alternatives considered
- **Domain-specific services/branches** — rejected: doesn't scale past two
  domains, and defeats the "same platform across HRMS/Campaigns/Gathr" thesis
  from the original design doc.

## Consequences
Target: a new domain configured in under a week, mostly spent on seeds and
golden questions, not plumbing (Use Cases doc §6.5) — that ratio is the
actual portability claim, and it's measurable once Phase 1 ships the first
real pack.
