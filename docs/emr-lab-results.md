# EMR Lab Results

Use this file to record outcomes from `docs/EMR_HANDS_ON_LAB_PLAN.md`.

## 1) Context

### Project
- Repo: `prediction-market-analysis`
- Job under test: `emr/jobs/kalshi_volume_quarterly.py`
- Plan reference: `docs/EMR_HANDS_ON_LAB_PLAN.md`

### Environment
- AWS account ID: `<ACCOUNT_ID>`
- AWS region: `<REGION>`
- S3 bucket: `s3://<BUCKET>`
- EMR Serverless app ID: `<APP_ID>`
- Runtime role: `<ROLE_ARN>`
- Date range tested: `<START_DATE>` to `<END_DATE>`

### Dataset profile
- Input path: `s3://<BUCKET>/raw/kalshi/trades/`
- Approx input size (GB): `<VALUE>`
- Approx row count: `<VALUE>`
- Sampling strategy: `<LIMIT / date filter / full>`

## 2) Success Criteria

Mark pass/fail with evidence links.

| Criterion | Target | Result | Evidence |
|---|---|---|---|
| Spark job completes successfully | Yes | `<PASS/FAIL>` | `<CloudWatch/S3/log reference>` |
| Curated parquet written | Yes | `<PASS/FAIL>` | `s3://<BUCKET>/curated/kalshi/volume_quarterly/` |
| Athena query returns expected rows | Yes | `<PASS/FAIL>` | `<query result link/screenshot>` |
| Step Functions workflow succeeds | Yes | `<PASS/FAIL>` | `<execution ARN>` |
| Scheduled run succeeds (EventBridge) | Yes | `<PASS/FAIL>` | `<schedule/execution evidence>` |

## 3) Run Log

One row per actual job run.

| Run # | Date (UTC) | Platform | Input scale | Config profile | Duration (min) | Status | Notes |
|---|---|---|---|---|---:|---|---|
| 1 | `<YYYY-MM-DD>` | `<Serverless/EC2>` | `<sample/full>` | `<baseline>` | `<value>` | `<success/fail>` | `<notes>` |
| 2 |  |  |  |  |  |  |  |
| 3 |  |  |  |  |  |  |  |

## 4) Experiment Matrix (Lab 6)

Change one variable at a time where possible.

| Exp ID | Changed setting | Value | Other settings fixed? | Data scale | Runtime (min) | Approx cost (USD) | Shuffle/Skew observations | Outcome |
|---|---|---|---|---|---:|---:|---|---|
| E1 | `spark.executor.memory` | `<e.g. 4g>` | `<yes/no>` | `<sample/full>` | `<value>` | `<value>` | `<notes>` | `<better/same/worse>` |
| E2 | `spark.executor.cores` |  |  |  |  |  |  |  |
| E3 | `spark.dynamicAllocation.maxExecutors` |  |  |  |  |  |  |  |
| E4 | Input scale | `<5x>` | `<yes/no>` | `<5x>` |  |  |  |  |

## 5) Spark Configuration Profiles

Record full profiles used in tests.

### Baseline profile
```text
<paste sparkSubmitParameters baseline>
```

### Profile A
```text
<paste sparkSubmitParameters profile A>
```

### Profile B
```text
<paste sparkSubmitParameters profile B>
```

## 6) Performance Findings

### Stage-level observations
- Longest stage(s): `<stage IDs or names>`
- Main bottleneck: `<shuffle / skew / IO / serialization / startup>`
- Spill observed: `<yes/no + details>`
- Data skew observed: `<yes/no + keys if known>`

### Practical notes
- What improved runtime the most:
- What reduced cost the most:
- What change had no meaningful impact:

## 7) Cost Findings

Track both compute and query costs where possible.

| Component | Metric | Value | Source |
|---|---|---:|---|
| EMR Serverless | Total job cost | `<USD>` | `<billing calc / estimate method>` |
| EMR on EC2 | Cluster + step cost | `<USD>` | `<billing calc / estimate method>` |
| Athena | Query scan cost | `<USD>` | `<Athena console>` |
| S3 | Storage + requests (if tracked) | `<USD>` | `<S3 cost explorer>` |

Cost assumptions:
- Pricing snapshot date: `<YYYY-MM-DD>`
- Estimation method: `<AWS pricing calculator / console / rough estimate>`

## 8) Serverless vs EC2 Comparison (Lab 5)

| Dimension | EMR Serverless | EMR on EC2 | Winner for this repo |
|---|---|---|---|
| Setup speed | `<notes>` | `<notes>` | `<Serverless/EC2/Tie>` |
| Ops overhead |  |  |  |
| Runtime predictability |  |  |  |
| Cost at current scale |  |  |  |
| Cost at projected scale |  |  |  |
| Debuggability |  |  |  |
| Team skill fit |  |  |  |

## 9) Final Recommendation

### Decision
- Recommended primary platform: `<EMR Serverless / EMR on EC2>`
- Confidence level: `<Low/Medium/High>`
- Effective date: `<YYYY-MM-DD>`

### Why
1. `<reason 1>`
2. `<reason 2>`
3. `<reason 3>`

### Risks and mitigations
| Risk | Impact | Mitigation |
|---|---|---|
| `<risk>` | `<impact>` | `<mitigation>` |

## 10) Next Actions

1. `<action>`
2. `<action>`
3. `<action>`

## Appendix A: Useful IDs and Links

- Step Functions state machine ARN: `<ARN>`
- Last successful execution ARN: `<ARN>`
- CloudWatch log group: `<LOG_GROUP>`
- Athena workgroup: `<WORKGROUP>`
- Glue database/table: `<DB.TABLE>`

## Appendix B: Repro Commands

```bash
# Upload Spark job
aws s3 cp emr/jobs/kalshi_volume_quarterly.py s3://<BUCKET>/code/emr/jobs/

# Submit EMR Serverless job
aws emr-serverless start-job-run \
  --cli-input-json file://emr/templates/start-job-run-kalshi-volume.json

# Validate output
aws s3 ls s3://<BUCKET>/curated/kalshi/volume_quarterly/
```
