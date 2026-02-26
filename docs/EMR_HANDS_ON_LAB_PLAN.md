# EMR Hands-On Lab Plan (For `prediction-market-analysis`)

## Purpose
This plan uses this repository as a realistic project to learn AWS big data tooling, especially EMR.  
You will build a full pipeline from parquet inputs to scheduled Spark jobs and SQL analytics.

Target architecture:

`repo/local parquet -> S3 raw -> EMR Serverless Spark -> S3 curated -> Glue Catalog -> Athena -> Step Functions/EventBridge`

## What You Will Learn
By the end, you should be able to:

1. Explain when to use EMR Serverless vs EMR on EC2.
2. Run Spark jobs on EMR against S3 parquet data.
3. Tune Spark resource settings and understand performance/cost tradeoffs.
4. Publish curated data to Glue and query it via Athena.
5. Orchestrate EMR jobs with Step Functions and schedule them.
6. Produce a short engineering decision memo based on measured results.

## Time Budget
- Core path: 8 to 12 hours total.
- Stretch goals: +3 to 5 hours.

## Prerequisites
1. AWS account with permissions for S3, IAM, EMR Serverless, Glue, Athena, Step Functions, CloudWatch, EventBridge.
2. AWS CLI configured (`aws configure`).
3. Local repo setup completed:

```bash
uv sync
```

4. Optional but recommended local data:
- `make setup` (downloads the full dataset), or a smaller custom sample.

## Repository Assets Used
1. Local analysis reference: `src/analysis/kalshi/volume_over_time.py`
2. EMR Spark job: `emr/jobs/kalshi_volume_quarterly.py`
3. EMR job payload template: `emr/templates/start-job-run-kalshi-volume.json`
4. Step Functions template: `emr/templates/stepfunctions-emr-serverless-kalshi-volume.json`

## Tracking Checklist
- [ ] Lab 0 complete
- [ ] Lab 1 complete
- [ ] Lab 2 complete
- [ ] Lab 3 complete
- [ ] Lab 4 complete
- [ ] Lab 5 complete
- [ ] Lab 6 complete
- [ ] Final results documented in `docs/emr-lab-results.md`

---

## Lab 0 (60 min): Baseline + AWS Setup

### Learning goals
1. Confirm local + AWS environment readiness.
2. Establish consistent naming and reusable environment variables.
3. Create minimum IAM boundaries for EMR Serverless runtime.

### Steps
1. Export baseline variables:

```bash
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET=pma-emr-lab-$AWS_ACCOUNT_ID-$AWS_REGION
export EMR_APP_NAME=pma-emr-serverless
```

2. Create S3 bucket:

```bash
aws s3 mb s3://$BUCKET --region $AWS_REGION
```

3. Create S3 folder structure:

```bash
aws s3api put-object --bucket $BUCKET --key raw/kalshi/trades/
aws s3api put-object --bucket $BUCKET --key curated/kalshi/volume_quarterly/
aws s3api put-object --bucket $BUCKET --key code/emr/jobs/
aws s3api put-object --bucket $BUCKET --key logs/emr-serverless/
```

4. Create IAM runtime role for EMR Serverless:
- Trust principal: `emr-serverless.amazonaws.com`
- Access scope:
- Read from `s3://$BUCKET/raw/*` and `s3://$BUCKET/code/*`
- Write to `s3://$BUCKET/curated/*` and `s3://$BUCKET/logs/*`
- Optional Glue permissions for catalog write/read

### Done when
1. Bucket and prefixes exist.
2. Runtime role ARN is available.
3. `aws sts get-caller-identity` returns expected account.

### Reflection
1. Why should runtime role permissions be prefix-scoped instead of bucket-wide?
2. What extra permissions are needed if the job writes Glue tables directly?

---

## Lab 1 (60 to 90 min): Prepare Repo Data Sample for S3

### Learning goals
1. Build a reproducible sample dataset for fast iteration.
2. Validate schema assumptions before distributed processing.

### Steps
1. Create a local sample parquet from Kalshi trades:

```bash
uv run python - <<'PY'
import duckdb, pathlib
pathlib.Path("tmp").mkdir(exist_ok=True)
duckdb.sql("""
COPY (
  SELECT * FROM 'data/kalshi/trades/*.parquet'
  LIMIT 2000000
) TO 'tmp/kalshi_trades_sample.parquet' (FORMAT PARQUET)
""")
print("wrote tmp/kalshi_trades_sample.parquet")
PY
```

2. Upload sample to S3:

```bash
aws s3 cp tmp/kalshi_trades_sample.parquet s3://$BUCKET/raw/kalshi/trades/
```

3. Validate object landed:

```bash
aws s3 ls s3://$BUCKET/raw/kalshi/trades/
```

### Done when
1. `s3://$BUCKET/raw/kalshi/trades/` contains sample parquet.
2. Local and S3 sample counts are known and documented.

### Reflection
1. Why sample first before testing large Spark jobs?
2. What bias can `LIMIT` sampling introduce?

---

## Lab 2 (90 min): First EMR Serverless Spark Job

### Learning goals
1. Understand EMR Serverless application lifecycle.
2. Submit a Spark batch job with resource settings.
3. Capture logs and outputs for debugging.

### Steps
1. Upload job code:

```bash
aws s3 cp emr/jobs/kalshi_volume_quarterly.py s3://$BUCKET/code/emr/jobs/
```

2. Create EMR Serverless app (cost-constrained):

```bash
aws emr-serverless create-application \
  --name $EMR_APP_NAME \
  --type SPARK \
  --release-label emr-7.12.0 \
  --maximum-capacity cpu=16vCPU,memory=64GB,disk=200GB \
  --scheduler-configuration maxConcurrentRuns=1,queueTimeoutMinutes=30
```

3. Fill placeholders in `emr/templates/start-job-run-kalshi-volume.json`.
4. Submit job:

```bash
aws emr-serverless start-job-run \
  --cli-input-json file://emr/templates/start-job-run-kalshi-volume.json
```

5. Monitor job state:

```bash
aws emr-serverless get-job-run \
  --application-id <APP_ID> \
  --job-run-id <JOB_RUN_ID>
```

### Done when
1. Output exists at `s3://$BUCKET/curated/kalshi/volume_quarterly/`.
2. CloudWatch/S3 logs are accessible.
3. You can explain each Spark parameter in the payload.

### Common failure modes
1. `AccessDenied`: runtime role missing S3 prefix permissions.
2. `ValidationException`: bad app ID, release label, or JSON field shape.
3. Job succeeds but no output: wrong input path or empty source data.

### Reflection
1. Which settings matter most for this aggregation workload and why?
2. What indicates under-provisioned executors in logs/UI?

---

## Lab 3 (45 to 60 min): Glue + Athena

### Learning goals
1. Register curated S3 data in the catalog.
2. Query Spark output with Athena SQL.

### Steps
1. Create Glue database (example name: `pma_lab`).
2. Create crawler for `s3://$BUCKET/curated/kalshi/volume_quarterly/`.
3. Run crawler and verify table creation.
4. Query in Athena:

```sql
SELECT * FROM pma_lab.volume_quarterly ORDER BY quarter;
```

### Done when
1. Athena returns valid quarterly rows.
2. Table schema matches expected Spark output columns.

### Reflection
1. When should you use crawler discovery vs explicit schema DDL?
2. What partitioning strategy helps future Athena costs?

---

## Lab 4 (60 to 90 min): Orchestration with Step Functions

### Learning goals
1. Run EMR jobs with workflow-level reliability.
2. Handle success/failure and app stop logic safely.
3. Move from ad hoc runs to scheduled operations.

### Steps
1. Create state machine from `emr/templates/stepfunctions-emr-serverless-kalshi-volume.json`.
2. Provide execution input with:
- `applicationId`
- `executionRoleArn`
- `entryPoint`
- `entryPointArguments`
- `sparkSubmitParameters`
- `logUri`, `logGroupName`, `logStreamNamePrefix`
3. Execute once manually.
4. Add EventBridge Scheduler for daily trigger.

### Done when
1. One full workflow run succeeds end-to-end.
2. Failure path also stops app and surfaces clear error status.

### Reflection
1. Why is `.sync` integration useful for batch orchestration?
2. What retry policy would you add for transient failures?

---

## Lab 5 (90 to 120 min): EMR on EC2 Comparison

### Learning goals
1. Understand operational and economic differences between EMR modes.
2. Practice cluster-based execution and step submission.

### Steps
1. Launch EMR on EC2 cluster with Spark.
2. Submit the same job via `spark-submit` step.
3. Capture metrics:
- Runtime
- Approximate cost
- Setup complexity
- Operational overhead (startup, shutdown, tuning effort)

### Done when
1. Side-by-side comparison table exists in your notes.
2. You write a short recommendation for this workload type.

### Reflection
1. At what throughput does EC2 become more attractive than Serverless?
2. Which team skills are required to operate EC2 clusters well?

---

## Lab 6 (45 min): Performance + Cost Tuning

### Learning goals
1. Build practical Spark tuning intuition.
2. Connect configuration changes to measurable outcomes.

### Experiments
1. Vary executor size:
- `spark.executor.memory`
- `spark.executor.cores`
2. Vary autoscaling ceiling:
- `spark.dynamicAllocation.maxExecutors`
3. Increase data volume by 5x.

### Metrics to track
1. End-to-end duration.
2. Shuffle-heavy stage behavior (spill/skew signs).
3. Approximate cost per run.

### Done when
1. `docs/emr-lab-results.md` includes:
- Test matrix
- Results table
- Final recommended settings for this job

### Reflection
1. Which config gave the best cost/performance ratio?
2. What is your first tuning move when jobs spill heavily?

---

## Cost and Safety Guardrails
1. Keep Serverless max capacity low early.
2. Keep `maxConcurrentRuns=1` while learning.
3. Start with sampled data.
4. Stop/delete unused apps and terminate EC2 clusters after labs.
5. Set AWS Budget alarms before running large experiments.

## Suggested Deliverables
1. `docs/emr-lab-results.md`
2. `docs/emr-architecture-notes.md` with a final architecture diagram
3. `docs/emr-decision-record.md` choosing Serverless or EC2 for this repo

## Stretch Goals
1. Port one additional analysis to Spark (example: calibration by price).
2. Convert curated output to partitioned layout and measure Athena scan reduction.
3. Add basic data quality checks before write (null checks, range checks).
4. Add CI job that validates EMR job template JSON syntax.
