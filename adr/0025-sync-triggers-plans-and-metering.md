# 0025. Sync triggers, plans and metering

Date: 2026-10-15
Status: **Accepted** 2026-10-19 — Sync now, the plan check and the usage ledger are built (`cce.api_sync`, the console at `/demo/manage`); event and scheduled triggers wait for an export API from EazeWork (design: [design/09](../design/09-roles-plans-and-settings.md) §2, §7.3)

## Context
EazeWork wants syncs to be event-driven with a "Sync now" button, and scheduled syncs (weekly or
monthly) as a paid add-on, with some syncs a month free (e.g. 4) and the rest paid.

## Decision
1. Three triggers (event, Sync now, schedule) feed the existing sync run (ADR-0022) through a
   **plan check**, then a **debounce** (events only; default 15 minutes) and a **fetch** of the
   export (EazeWork API; a drop folder until then).
2. A **plan** per company, set by EazeWork: included syncs per period, which triggers count,
   whether no-change runs count, what happens over the limit (`block` / `ask` / `allow`),
   schedule add-on, event rate limit. These are values, not code, so pricing changes need no
   release.
3. A **usage ledger** records every run with its trigger and whether it counted. CCE meters;
   EazeWork bills from a monthly usage report. CCE never stores prices.
4. Recommended counting rule (EazeWork to confirm): requested syncs (Sync now, schedule) count;
   change events are free within a fair-use rate limit; runs that find no change are free.

## Implementation (2026-10-19)
- `POST /sync/upload` is "Sync now": the Tool Administrator's upload becomes one sync run. The
  run carries `trigger: sync_now` and `counted`; `GET /sync/usage` is the month's ledger.
- Plan values are settings EazeWork sets at platform level (`plan.syncs_included`,
  `plan.over_limit` = block | ask | allow, `plan.count_no_change`); a company reads them, can't
  change them. "ask" is a 402 that the console turns into a confirmation.
- Raw uploads are deleted by the worker after `CCE_EXPORT_RETENTION_DAYS` (30).
- Not built: event and scheduled triggers, the debounce, the fetch from an EazeWork API — all
  need the export API they haven't got yet (design 09 §7.2, re-ask 2).

## Consequences
- Answers stay correct after every real change without anyone paying for correctness.
- Event syncs depend on EazeWork providing an export API.
