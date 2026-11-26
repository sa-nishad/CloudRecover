# Measuring RTO and RPO

CloudRecover has demonstrated a working regional recovery path, but end-to-end RTO and RPO have not yet been formally measured. This document defines how to measure them so results are comparable between drills. It is a measurement plan only; it does not contain results.

## Definitions

- **Outage start (T0):** the moment the primary application path stops serving requests. For a drill, record the exact time the failure is injected.
- **Detection (T1):** the time the Route 53 health check marks the primary endpoint unhealthy and the CloudWatch alarm enters the ALARM state.
- **Workflow dispatched (T2):** the time the recovery GitHub Actions workflow run is created by the Lambda function.
- **Database available (T3):** the time the restored RDS instance in eu-west-2 reports available.
- **Application ready (T4):** the first successful response from /health/ready on the recovery ALB.
- **Clients recovered (T5):** the first successful request through the normal API hostname that is served by the recovery environment.

**RTO** for a drill is T5 minus T0. The intermediate timestamps show where time is spent. The under-one-minute workflow duration seen in earlier trigger tests covers only part of this interval and must not be reported as the RTO.

**RPO** is the amount of data written before T0 that is absent after recovery. It is determined by comparing known writes with restored data, not inferred from snapshot or workflow timing.

## Drill procedure

1. Record the AWS account, regions, Terraform commit and the snapshot identifier expected to be used.
2. Before injecting the failure, write a sequence of uniquely identifiable stock movements to the primary API, each with a client-side timestamp, and save the list of identifiers.
3. Note the age of the most recent eligible RDS snapshot.
4. Inject the failure and record T0. The existing method of removing the application-to-database ingress rule is suitable and does not delete data.
5. Capture T1 to T5 from the CloudWatch alarm history, the GitHub Actions run log, the RDS event log and a client-side polling loop against the API hostname.
6. After recovery, query the recovered database for the recorded identifiers and count how many are missing.
7. Restore the primary environment to its normal state before ending the drill, and review the Terraform plan to confirm there is no unintended drift.

## Results template

| Field | Value |
| --- | --- |
| Drill date | |
| Terraform commit | |
| Snapshot used and its age at T0 | |
| T0 to T1 (detection) | |
| T1 to T2 (dispatch) | |
| T2 to T3 (database restore) | |
| T3 to T4 (application ready) | |
| T4 to T5 (DNS and client convergence) | |
| End-to-end RTO (T5 minus T0) | |
| Writes recorded / writes missing after recovery (RPO evidence) | |
| Notes and anomalies | |

## Caveats

- DNS caching and existing client connections can delay T5 for some clients. Poll from more than one network if possible.
- Snapshot-based recovery cannot recover writes made after the snapshot was taken, so a measured RPO is expected to be bounded by snapshot frequency.
- Repeat the drill several times and report the range rather than a single figure.
