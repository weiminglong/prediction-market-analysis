# EMR Architecture Notes

Working notes for the EMR learning implementation based on:
- `docs/EMR_HANDS_ON_LAB_PLAN.md`
- `docs/emr-lab-results.md`

## 1) Scope

### Objective
- Build and operate a reproducible data pipeline for this repo on AWS.

### In scope
1. Batch Spark processing with EMR.
2. S3-based data lake layout.
3. Glue/Athena queryability.
4. Step Functions + EventBridge orchestration.

### Out of scope
1. Real-time streaming ingestion.
2. Multi-region replication.
3. Cross-account data sharing.

## 2) High-Level Architecture

```text
Local/Repo Parquet
    ->
S3 Raw Zone (s3://<BUCKET>/raw/...)
    ->
EMR Spark Job (Serverless or EC2)
    ->
S3 Curated Zone (s3://<BUCKET>/curated/...)
    ->
Glue Data Catalog
    ->
Athena Queries
    ->
Step Functions (start app -> run job -> stop app)
    ->
EventBridge Scheduler (daily trigger)
```

## 3) Data Zones and Naming

### Raw zone
- Prefix: `s3://<BUCKET>/raw/kalshi/trades/`
- Characteristics: immutable input snapshots, minimal transformation.

### Curated zone
- Prefix: `s3://<BUCKET>/curated/kalshi/volume_quarterly/`
- Characteristics: typed and query-ready output from Spark jobs.

### Logs
- Prefix: `s3://<BUCKET>/logs/emr-serverless/`
- CloudWatch group: `/aws/emr-serverless/prediction-market-analysis`

## 4) Compute Design

### Primary path
- EMR Serverless Spark job:
- Script: `emr/jobs/kalshi_volume_quarterly.py`
- Payload template: `emr/templates/start-job-run-kalshi-volume.json`
- Initial stance: use EMR Serverless as default execution mode for this repo while data volume is moderate and team size is small.

### Comparison path
- EMR on EC2 using equivalent Spark step for benchmarking.
- Initial stance: use as a benchmark and fallback path, not default.

### Runtime assumptions
1. Input is parquet in S3.
2. Job is batch and idempotent with overwrite/append mode control.
3. Data volume may increase; scaling is handled with dynamic allocation tuning.

## 5) Orchestration Design

### Workflow
1. Start EMR Serverless application.
2. Submit Spark job run.
3. Stop application on success or failure.

Template:
- `emr/templates/stepfunctions-emr-serverless-kalshi-volume.json`

### Schedule
- EventBridge Scheduler triggers Step Functions daily.
- Initial stance: one run per day with `maxConcurrentRuns=1`.

## 6) Security and Access

### IAM runtime role principles
1. Prefix-scoped S3 permissions.
2. Least privilege for Glue and CloudWatch APIs.
3. Separate role for orchestration (Step Functions) if needed.

### Data protection
1. Enable default S3 encryption.
2. Use bucket policies to restrict public access.
3. Consider KMS CMKs for stricter key control.

## 7) Observability

### Signals to capture
1. Job status and duration.
2. Spark stage bottlenecks (shuffle/skew/spill).
3. Data output row counts and freshness.
4. Failure reasons by category (IAM, config, data, infra).

### Dashboards and logs
1. CloudWatch Logs for driver/executor output.
2. S3 log sink for EMR Serverless artifacts.
3. Optional custom metrics in CloudWatch for SLA tracking.

## 8) Reliability

### Failure handling
1. Stop app in both success and failure paths.
2. Retry transient failures with bounded retry policy.
3. Surface terminal failures through Step Functions execution status.

### Idempotency
1. Use deterministic output paths per run/date when needed.
2. Prefer atomic write patterns and explicit write mode.

## 9) Cost Controls

1. Start with sampled data.
2. Keep Serverless max capacity conservative.
3. Tune dynamic allocation before increasing executor size.
4. Partition curated output for Athena scan reduction.
5. Set AWS Budgets and alerts.

## 10) Initial Recommended Stance (Before Benchmark Results)
1. Compute mode: EMR Serverless first.
2. Curated layout: parquet partitioned by `year` for the quarterly aggregate job.
3. Schema management: hybrid model.
- Use Glue crawler during exploration.
- Move critical production tables to explicit DDL once schema stabilizes.
4. Orchestration: Step Functions + EventBridge Scheduler.
5. Concurrency and safety:
- Keep one daily run, no overlap.
- Keep bounded max capacity and tune only after baseline metrics.

## 11) Open Questions

1. Should curated outputs be partitioned by `year` only or `year/quarter`?
2. Should Glue schema be crawler-managed or explicit DDL-managed?
3. What run frequency provides best freshness/cost ratio?
4. At what scale does EC2 become cheaper than Serverless for this workload?

## 12) Decisions Log (Pointer)

Record final decisions in:
- `docs/emr-decision-record.md`

## 13) Diagram Placeholder

Replace this section with your final architecture diagram (draw.io, Mermaid, or screenshot link).

```text
[Architecture diagram goes here]
```
