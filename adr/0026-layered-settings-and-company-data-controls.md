# 0026. Layered settings, and company-level data controls instead of per-company backups

Date: 2026-10-15
Status: **Accepted — layered settings implemented 2026-10-15** (migration 0010; `cce.company_settings`;
`GET/PUT/DELETE /settings`). Company data controls (restore to date, retention, export): not yet built.
Design: [design/09](../design/09-roles-plans-and-settings.md) §3

## Context
EazeWork wants to configure each client first (e.g. backups, restore policy) and let the client
adjust within limits. ADR-0008 put product facts in the database and deployment facts in env, but
there is no per-company or per-platform settings store. A database backup holds every company.

## Decision
1. **Three levels**: installation (env, operator) → platform (EazeWork defaults, per-company
   overrides, locks) → company (HR admin, within bounds). Each setting is declared once with its
   levels, default, allowed values and lockability. Changes are versioned and logged.
2. `CCE_STALE_INTERPRETATIONS`, `CCE_SYNC_HOLD_*` and `CCE_CORRECTIONS_REQUIRE_APPROVAL` move to
   this store; their env values become installation defaults.
3. **Server backups stay with the operator** (ADR-0007). Restoring a database affects every
   company, so no product role can do it.
4. **Per-company data controls** instead: roll back a sync (built), restore knowledge to a date,
   retention windows for questions/answers and raw exports, a monthly company export, and
   support access. EazeWork sets defaults and limits; HR admins choose within them.

## Consequences
- "Backup and restore per company" is delivered as restore-to-date and export, which only need
  versioned knowledge (already there), not separate databases.
- Retention windows also cover DPDP obligations for employee questions.

## Implementation notes (2026-10-15)

- Settings declared in `cce.company_settings.SPECS` with kind, allowed values, the installation
  default (env) and what a company may do: `any`, `lower_only` (stricter only) or `none`.
- Levels in `settings_values`: `platform` (EazeWork default for every company), `company_override`
  (EazeWork's value or lock for one company), `company`. Every write goes to `settings_history`.
- Settings in effect today: `sync.stale_interpretations`, `sync.hold_fraction` (stricter only),
  `sync.hold_removals` (stricter only), `corrections.require_approval`, `access.account_managers`.
  The sync, the dependents handling, the worker fan-out and corrections read the company's value.
- `access.*` settings are always the company's decision: EazeWork can set a default but can't lock it.
- New settings (retention, schedules, plans) are added to `SPECS` as their features are built.

