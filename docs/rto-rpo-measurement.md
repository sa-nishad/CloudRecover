# Measuring RTO and RPO

CloudRecover has demonstrated a working regional recovery path, but end-to-end RTO and RPO have not yet been formally measured. This document defines how to measure them so results are comparable between drills. It is a measurement plan only; it does not contain results.

## Definitions

- **Outage start (T0):** the moment the primary application path stops serving requests. For a drill, record the exact time the failure is injected, and also the time of the first failed request seen by the client-side polling loop. Use the injection time as T0 and report the gap between the two.
- **Detection (T1):** the time the CloudWatch alarm enters the ALARM state. Also record when the Route 53 health check first marks the primary endpoint unhealthy, because the gap between the two is part of detection time.
- **Workflow dispatched (T2):** the time the recovery GitHub Actions workflow run is created by the Lambda function.
- **Database available (T3):** the time the restored RDS instance in eu-west-2 reports available.
- **Application ready (T4):** the first successful response from /health/ready on the recovery ALB.
- **Clients recovered (T5):** the first successful request through the normal API hostname that is demonstrably served by the recovery environment (for example by a region or version identifier in the response), followed by consecutive successes so a single lucky response is not counted.

**RTO** for a drill is T5 minus T0. The intermediate timestamps show where time is spent. The under-one-minute workflow duration seen in earlier trigger tests covers only part of this interval and must not be reported as the RTO.

**RPO** is the amount of data written before T0 that is absent after recovery. It is determined by comparing known writes with restored data, not inferred from snapshot or workflow timing. Report it both as a count of missing writes and as a time: T0 minus the client-side timestamp of the newest recorded write that is present after recovery.

## Drill procedure

1. Record the AWS account, regions, Terraform commit and the snapshot identifier expected to be used.
2. Start a client-side polling loop against the API hostname that logs a timestamp, status code and the identifier of the environment that answered. Keep it running until after T5.
3. Before injecting the failure, write a sequence of uniquely identifiable stock movements to the primary API, each with a client-side timestamp, and save the list of identifiers. Keep writing at a steady rate up to the moment of injection so the newest writes fall close to T0.
4. Note the age of the most recent eligible RDS snapshot.
5. Inject the failure and record T0. The existing method of removing the application-to-database ingress rule is suitable and does not delete data.
6. Capture T1 to T5 from the CloudWatch alarm history, the Route 53 health check status, the GitHub Actions run log, the RDS event log and the polling loop.
7. After recovery, query the recovered database for the recorded identifiers and count how many are missing.
8. Restore the primary environment to its normal state, tear down or reset the recovery environment as the runbook requires, and review the Terraform plan to confirm there is no unintended drift.

## Results template

| Field | Value |
| --- | --- |
| Drill date and run number | |
| Terraform commit | |
| Snapshot used and its age at T0 | |
| Polling vantage points | |
| T0 to T1 (detection) | |
| T1 to T2 (dispatch) | |
| T2 to T3 (database restore) | |
| T3 to T4 (application ready) | |
| T4 to T5 (DNS and client convergence) | |
| End-to-end RTO (T5 minus T0) | |
| Writes recorded / writes missing after recovery | |
| RPO as time (T0 minus newest surviving write) | |
| Notes and anomalies | |

## Caveats

- DNS caching and existing client connections can delay T5 for some clients. Poll from more than one network if possible.
- Snapshot-based recovery cannot recover writes made after the snapshot was taken, so a measured RPO is expected to be bounded by snapshot frequency.
- Removing the database ingress rule leaves the primary compute running, so detection and failover behaviour may differ from a true regional outage. State this limitation when reporting results.
- Repeat the drill several times and report the range rather than a single figure.
