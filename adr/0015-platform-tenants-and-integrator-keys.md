# 0015. Platform tenants (shared read-only knowledge) and integrator keys

Date: 2026-10-09
Status: **Accepted — implemented** (migration 0005; 10 walking-skeleton checks)

## Context

EazeWork serves many customer companies from one product. Two kinds of knowledge
follow from that:

- **Per company:** its policies, its configuration, its corrections, its precedents.
  Strictly private — tenancy as ADR-0010 already enforces.
- **Per product:** EazeWork's help center (how leave deduction works, which report
  shows what). Identical for every company. Copying it into each tenant would mean
  N copies to re-ingest, re-certify and keep in sync.

And one integration fact: EazeWork's own backend will call CCE on behalf of whichever
company's user is asking. It should not have to hold one API key per customer.

## Decision

1. **Tenants have a kind:** `client` (a customer company) or `platform` (EazeWork).
2. **RLS is split by verb** on `knowledge_objects`, `entities`, `entity_aliases`,
   `dependency_edges`: a session may *read* its own rows and any platform tenant's
   rows; it may *insert/update* only its own. `cce.db.tenant_session` sets
   `app.shared_tenant_ids` per transaction from the tenants table (which the app role
   cannot read). Proposals, traces, gaps and the change log stay strictly per tenant.
   Verified at the database level, not only in application code: a client session's
   UPDATE of a platform row affects 0 rows and an INSERT as the platform is refused.
3. **Clients can't supersede or deprecate platform knowledge.** Certification refuses
   it with a pointer to the right tool: a `TenantOverride` that `interprets` the
   platform object, which is the precedence model the design already had.
4. **Own knowledge first.** Compilation ranks a company's own objects above platform
   ones of the same type; every citation carries `owner: own | platform`.
5. **Integrator keys.** A key with role `integrator`, owned by a platform tenant, may
   name the company a request is for in `X-CCE-Tenant` (id or exact name). Any other
   key naming another tenant gets 403. `/whoami` reports both identities.
6. **The asker's attributes** (employee type, level, location…) travel as `scope` on
   `/ask`; scope exclusion already filters rules meant for other attribute values.
7. **Opt-out:** `tenants.inherits_platform = false` hides platform knowledge from a
   tenant. The eval harness's tenants use it, so the golden set stays hermetic when a
   real platform tenant exists.

## Alternatives considered

- **Copy platform objects into every tenant** (what the pack loader does for statutory
  rows today). Simple, but N copies of a 100-article help center per release, and
  corrections to the help center would have to be repeated per company. Rejected for
  product docs; kept for now for the statutory fixtures (revisit: they are platform
  knowledge too).
- **A separate "shared" schema/table.** Duplicates every query path and the
  governance pipeline. Rejected: the shared layer is still certified knowledge and
  should go through the same proposal → certify loop, just in its own tenant.
- **Hierarchical tenants (parent/child).** More general than needed; one platform
  level is what EazeWork has. Revisit if a reseller layer appears.

## Consequences

- Help-center ingestion happens once, into the platform tenant; every company sees it.
- The integrator key is powerful — it can act for every company. It belongs only in
  EazeWork's backend secret store; company admins get ordinary keys.
- Lists (`GET /objects`, `GET /entities`) show the caller's own rows unless
  `include_platform=true`; answers include platform knowledge automatically.
- `cce.tenants` CLI: `list`, `create [--platform]`, `set-kind`, `key [--integrator]`.
