# Backup freshness and retention (proposal)

**Status: proposed and untested.** CloudRecover recovers the database from a cross-region RDS snapshot. The quality of that recovery depends on how recent the snapshot is and whether a usable one exists. This document proposes checks and a retention policy so that recovery-point freshness is explicit rather than assumed.

## Problem

The recovery workflow can select the latest eligible primary snapshot when none is specified. Nothing currently verifies that the selected snapshot is recent enough, so a stale snapshot would be restored silently and the data-loss window (RPO) could be far larger than expected.

## Proposed freshness check

Before restoring, the recovery workflow would:

1. Look up the selected snapshot and read its creation time.
2. Compute its age at the time the workflow starts.
3. Compare the age with a configured maximum, for example a workflow input or repository variable named MAX_SNAPSHOT_AGE_HOURS.
4. If the snapshot is older than the maximum, log a clear warning and either stop or require an explicit override input, depending on the chosen policy.
5. Write the snapshot identifier and age into the workflow summary so every recovery records which recovery point was used.

The maximum age should be derived from the RPO the project is willing to accept, which in turn should come from the measurements described in docs/rto-rpo-measurement.md. The value should be chosen by the project owner; this document does not set one.

## Proposed retention policy

- Keep enough recent automated snapshots in the primary region to cover the chosen freshness window plus a safety margin.
- Keep cross-region copies in eu-west-2 for the same period, and monitor that each copy completes.
- Keep a small number of older snapshots for a longer period so that a problem discovered late, such as silent data corruption, can still be recovered from.
- Never delete a snapshot that is currently referenced by an in-progress recovery.
- Review snapshot storage cost alongside the cost guidance in the README.

## Monitoring ideas

- A CloudWatch alarm or scheduled check that fires when no new cross-region snapshot copy has appeared within the freshness window.
- A periodic restore test of a recent snapshot into a throwaway instance to confirm that copies are actually restorable.

## Open questions

- Which maximum snapshot age is acceptable for this application?
- Should a stale snapshot block recovery, or recover with a loud warning, given that an outage is already in progress?
- Whether snapshots are taken by RDS automated backups, a scheduled job, or both, and how the cross-region copy is triggered today.
