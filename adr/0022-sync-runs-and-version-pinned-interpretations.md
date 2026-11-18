# 0022. Sync runs, version-pinned interpretations, and platform releases

Date: 2026-10-12
Status: **Accepted — implemented 2026-10-14** (migration 0009; `cce.sync`; CI: `scripts/check_sync.py`;
design: [architecture/04-change-paths.md](../architecture/04-change-paths.md))

## Context

EazeWork wants company configuration synced weekly and its own (platform) documentation
monthly. The readers already turn a re-import into a diff by source key (ADR-0017), but:

- An import is not recorded as a unit: no file hash, no reader version, no way to undo it.
- Our reader code changing and the company's configuration changing produce the same kind
  of diff, so a code release can look like HR rewrote their policies.
- On supersede, `certify_proposal` moves incoming `interprets` edges to the new version.
  A Clarification or TenantOverride written against the old rule silently applies to the new
  one and keeps ranking first.
- Rows that disappear from an export are never deprecated.
- Every config change waits in the queue as `risk=med` and is approved in bulk, which trains
  certifiers to rubber-stamp (Design Review G9).

## Decision

1. **`source_snapshots` table**: one row per import (tenant, source, taken_at, file hash,
   reader version, kind `sync | re-render`, status `staged | held | applied | discarded |
   rolled_back`, counts). Proposals carry `snapshot_id`. An identical file hash ends the run as
   "no change".
2. **Code and source changes never mix.** A sync must use the same reader version as the
   previous applied run of that source. A new reader version is applied by a **re-render run**
   over the previous snapshot file. In a re-render, a diff with equal `parameters` and
   different text applies automatically; a diff with different `parameters` is held.
3. **Routing for company config syncs**: new or changed objects with no human-authored
   dependents are certified automatically (`decided_by = sync:<snapshot>`); a changed object
   that a Clarification, TenantOverride or answered question interprets is certified and the
   dependent is **suspended** from answers until HR re-confirms it; removals become deprecate
   proposals for review; questions whose cause disappeared are closed.
4. **Size check**: a run that would change more than 20% of a company's objects, or deprecate
   more than a handful, is held as a whole for the operator.
5. **Pinned interpretations**: `dependency_edges.to_version` records the version an
   `interprets` edge was written against. Supersede no longer re-points those edges silently;
   it marks them for review (route 3 above). Other edge kinds keep today's behaviour.
6. **Platform releases**: imports into the platform tenant are snapshots with a release name,
   are never auto-certified, show cross-company impact as counts only, and put a review item in
   each affected company's Questions for HR. Company overrides of changed platform articles
   keep working but are flagged.
7. **Rollback by run**: undoing a run restores the previous version of every object it changed,
   re-activates what it deprecated and un-suspends what it suspended.
8. **Traces record `build_sha` and `prompt_hash`.**

## Alternatives considered

- **Keep re-pointing edges, add a warning.** Rejected: the stale Clarification would still rank
  first and answer confidently in the meantime.
- **Suspend nothing; let HR review every config diff.** Rejected: config is already the
  system of record, and reviewing hundreds of unchanged-in-meaning diffs a month teaches people
  to approve without reading.
- **Pin each company to a platform release.** Not now: EazeWork is SaaS, so every company runs
  the same product version. Revisit if versions or modules differ by company.

## Consequences

- One migration (snapshots table, `snapshot_id` on proposals, `to_version` on edges,
  `suspended` status or flag, trace columns) and changes to both readers, `certify_proposal`
  and the compiler (skip suspended objects).
- A sync becomes a schedulable job, which fits the existing queue (ADR-0002) and the event
  trigger in ADR-0023.
- Raw exports contain personal data (employee mapping). Proposed default: keep hash and
  parsed result; keep raw files 30 days. Needs EazeWork's agreement.

## Implementation notes (2026-10-14)

Where the build settled details the decision left open:

- **Reader version** is a fingerprint of the reader's own modules (`cce.sync.READERS`), not the
  git SHA, so unrelated code changes don't block syncs.
- **First import of a source is held**, so a person looks at a company once before its rules go
  live. A run whose items were all certified one by one (`cce.certify approve`, `./cce.sh
  approve`) is closed as applied when the next import starts, so the existing partner flow works.
- **Removals and re-confirmations are questions for HR** (`question_source` `removal` /
  `reconfirm`), answered with keep / retire (or new wording for a re-confirmation). They are never
  bulk-approved.
- **Platform changes**: the certifying session can't see companies' edges under RLS, so a
  `dependents_fanout` worker job (admin session) flags companies' own answers and overrides of a
  changed shared object. Flag, not suspend, as decided above.
- **`CCE_STALE_INTERPRETATIONS`**: `suspend` (default) | `flag` | `follow` (old behaviour). The
  choice between pausing and flagging is one of the questions put to EazeWork.
- **Integrator keys** can ask and propose, never certify, reject, answer questions or write
  objects. The image workflow publishes only after CI succeeds.
- **Review fixes (same day)**, from an independent review of the first commit:
  - Reader versions are fingerprints of the reader's *source*; the compiled image records them
    at build time (`cce/_reader_versions.py`), so moving between images doesn't block syncs.
    Help-center imports accept a reader change (their runs are always held anyway).
  - An HR answer about several rules stays pinned to each; one re-confirmation lists every
    change; "keep" and new wording attach to each rule's *current* version.
  - A partial EazeWork export (missing widgets) is held and never treated as removals or as
    resolving questions; a removal question is withdrawn if the rule comes back.
  - Rollback works after later "no change" runs (they're rolled back with it), refuses while a
    later run is held, re-points every edge to the restored version, reverses the shared-rule
    fan-out, and refuses once HR has answered one of its re-confirmations.
  - A held run with hand-certified items can't be discarded (apply, then roll back); a run
    whose items were all rejected is closed as discarded, so the export is proposed again.
  - Proposals can't carry the fields CCE sets on its own questions; integrator proposals are
    labelled `integrator:<key owner>` and skipped by bulk approval unless named.
- Not built: the weekly schedule itself (an operator or cron runs `ingest`), cross-company
  impact counts in a platform run's preview, and automatic regression runs after a sync.

