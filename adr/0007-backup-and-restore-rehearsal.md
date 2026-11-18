# 0007. Nightly pg_dump + offsite sync + rehearsed monthly restore

Date: 2026-08-10
Status: **Accepted — implemented 2026-10-14** (`deploy/backup.sh`; `./cce.sh backup | verify-backup | restore`;
`make backup | backup-verify`; CI and the partner-image gate dump and verify on every run)

## Context
The entire value proposition of this system is accumulated state. An
untested backup of a "cumulative" system is a time bomb — and a backup that's
never been restored is not a backup, it's a hope.

## Decision
Nightly `pg_dump -Fc` of the `cce-pgdata` volume, synced off-machine (restic
to any S3-compatible target), with a retention policy (7 daily / 4 weekly / 6
monthly) and a **rehearsed** monthly restore into a scratch container with a
row-count sanity check.

## Alternatives considered
- **Volume snapshot only, no logical dump** — rejected: doesn't survive a
  Postgres version upgrade cleanly; a logical dump does.
- **Skip rehearsal, trust the dump** — rejected explicitly: this is the
  Design Review's own warning (§2, G-something on backup rehearsal) and the
  cheapest insurance in the whole system.

## Consequences
**Action item, not yet done:** wire this up before real HRMS/domain content
exists — right now the `demo` tenant is disposable, so there's nothing to
protect yet, but that changes the moment Phase 1 starts.

## Implementation notes (2026-10-14)

- One portable script, `deploy/backup.sh` (bash 3.2-safe for the Mac Mini), used by the
  partner bundle (`./cce.sh`) and the repo (`make`). Postgres tools run inside the postgres
  container, so the host needs only docker, bash and awk.
- **Dump:** `pg_dump -Fc` streamed to `backups/cce-<date>-<time>.dump`, owner-only permissions,
  with a `.counts` file (row count per table) taken right after.
- **Retention:** newest dump per day for 7 days, per week (by day of month) for 4 weeks, per
  month for 6 months; safety dumps from restores are never pruned.
- **Off-site:** not restic, to avoid another dependency and repository to manage: an absolute
  folder (mounted drive, NAS, iCloud Drive) or any rclone remote (S3, R2, B2, Drive), optionally
  encrypted with openssl AES-256/PBKDF2 (`CCE_BACKUP_PASSPHRASE`). Choosing the target is still
  the operator's decision (DEPLOY-AND-DATA §5 item 3).
- **Rehearsal:** into a scratch *database* on the same server rather than a scratch container:
  it proves the dump restores and every table's row count matches, without needing a second
  Postgres. Tolerance: max(5 rows, 1%) per table, for writes during the dump.
- **Restore:** safety dump, stop api/worker, recreate the database, create the `cce_app` role if
  missing (new server), restore, start (migrations run). Tested onto a fresh Postgres cluster:
  the app role logs in and row-level security still holds.

