# 0008. Env holds deployment facts; DB holds product facts

Date: 2026-08-10
Status: Accepted — implemented in Phase 0 (`cce.settings.Settings`)

## Context
Configuration sprawl is a common failure mode: if "add a tenant" or "change a
risk policy" requires a redeploy, the governance model (proposals, certified
changes) is undermined by a side door.

## Decision
Environment variables hold **deployment facts only**: DB URL, LLM keys,
provider selection, log level, external base URL. Everything else — tenants,
domains, risk policies, model routing rules, feature flags per tenant — lives
in the database and is reachable through the normal API/proposal surface.

## Alternatives considered
- **Everything in env/config files** — rejected: makes multi-tenancy and
  per-domain configuration require redeploys, which contradicts the whole
  premise of governed, live-updatable knowledge.

## Consequences
Adding a tenant is an API call, not a deploy. Domain-pack install (ADR 0012)
follows the same rule: packs are source-of-truth in git but materialize into
the DB via proposals, never read from YAML at request time.
