# Failback procedure (proposal)

**Status: proposed and untested.** CloudRecover currently demonstrates recovery into the secondary region (eu-west-2). A controlled failback to the primary region (eu-west-1) is not implemented. This document sets out the constraints and a candidate procedure so it can be reviewed, implemented and drilled.

## Why failback is not just a DNS change

After recovery, the application accepts writes against the restored database in eu-west-2. The original primary database in eu-west-1 does not contain those writes. Pointing DNS back at the primary without first synchronising data would discard every write made during the recovery period.

## Preconditions

- The cause of the original primary outage is understood and resolved.
- The primary environment passes its own checks: infrastructure matches the Terraform configuration and /health/ready returns success through the primary ALB.
- A maintenance window is agreed, because the candidate procedure includes a short write freeze.
- A rollback decision point is defined before starting.

## Candidate procedure

1. **Verify the primary.** Run terraform plan in infrastructure/primary and confirm there is no unexpected drift. Confirm the primary application is healthy but not receiving production traffic.
2. **Freeze writes in the recovery region.** Stop the recovery application, or place it in a read-only or maintenance mode, so no new writes occur during the copy. Record the time of the freeze.
3. **Snapshot the recovery database.** Take a manual RDS snapshot of the recovery database after the freeze and wait for it to become available.
4. **Copy and restore in the primary region.** Copy the snapshot to eu-west-1 and restore it as the primary database, replacing or repointing the stale primary database according to the Terraform configuration.
5. **Validate.** Start the primary application against the restored database. Check /health/live and /health/ready through the primary ALB, and compare row counts or recent record identifiers with the recovery database.
6. **Cut over DNS.** Point the API hostname back at the primary ALB. Allow for DNS caching, and keep the recovery environment available until traffic has converged.
7. **Re-arm detection.** Confirm the Route 53 health check, CloudWatch alarm, SNS topic and Lambda trigger are healthy and pointing at the primary endpoint again.
8. **Retire or park the recovery environment.** After an agreed stabilisation period, scale down or destroy the recovery resources following the cost and cleanup guidance in the README. Retain the snapshots used.

## Rollback

If validation fails at step 5, leave DNS on the recovery environment, resume writes there and investigate. Because DNS has not changed, no data is lost. After step 6, rolling back requires repeating the data copy in the opposite direction.

## Write fencing

During steps 2 to 6 only one region should accept writes. The README lists write fencing as a future improvement. Until it exists, enforce this manually and record who confirmed that the recovery application is stopped before the copy begins.

## Open questions

- Whether to restore into a new database instance and swap, or to restore over the existing primary resources, and how each interacts with Terraform state.
- How long a write freeze the application's users can tolerate, which should be informed by the measured RTO and RPO.
- Whether to automate this as a separate GitHub Actions workflow with a manual approval gate.
