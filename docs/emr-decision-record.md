# EMR Decision Record

ADR-style record for platform and architecture decisions during the EMR labs.

Related docs:
- `docs/EMR_HANDS_ON_LAB_PLAN.md`
- `docs/emr-lab-results.md`
- `docs/emr-architecture-notes.md`

## ADR-001: Primary EMR Compute Mode for This Repo

### Status
- Proposed

### Date
- 2026-02-12

### Decision Owner
- Data Engineering Owner (Weiming)

### Context
This repository runs batch analytics over parquet datasets.  
We need a primary AWS execution model that balances:
1. Delivery speed.
2. Operational overhead.
3. Runtime performance.
4. Cost at current and projected scales.

### Options Considered
1. EMR Serverless
2. EMR on EC2

### Decision
- Selected option: `EMR Serverless` (default path)

### Rationale
1. Lowest operational overhead for a batch-first workflow.
2. Fastest path to production-like orchestration while learning core Spark concepts.
3. Better fit for bursty runs and iterative tuning on sampled data.

### Evidence
Reference measured results from `docs/emr-lab-results.md`:
1. Runtime comparison: pending benchmark completion (Lab 5 and Lab 6).
2. Cost comparison: pending benchmark completion (Lab 5 and Lab 6).
3. Operability comparison: expected lower ops overhead than EC2 for current scope.

### Consequences

#### Positive
1. Reduced infrastructure management and faster onboarding.
2. Easier scaling experiments via job-level Spark parameters.

#### Negative
1. Less low-level cluster control than EMR on EC2.
2. Potentially higher cost at very large or steady-state workloads.

### Risk Mitigations
| Risk | Mitigation | Owner |
|---|---|---|
| Serverless cost growth with scale | Re-evaluate with EC2 benchmark at defined thresholds | Data Engineering Owner |
| Runtime variance under heavy load | Pin key Spark configs and track p95 runtime | Data Engineering Owner |

### Review Trigger
Revisit this decision if:
1. Data volume per run exceeds 500 GB.
2. End-to-end runtime SLA tightens below 30 minutes consistently.
3. Monthly EMR cost exceeds planned budget by 20% or more.

---

## ADR-002: Data Layout for Curated Outputs

### Status
- Proposed

### Date
- 2026-02-12

### Context
Athena query costs and performance depend heavily on S3 layout and partitioning.

### Options Considered
1. Unpartitioned parquet output.
2. Partition by `year`.
3. Partition by `year` and `quarter`.

### Decision
- Selected option: Partition by `year` for the quarterly aggregate output.

### Rationale
1. Simple partition strategy with low write complexity.
2. Sufficient for current aggregate granularity and expected query patterns.

### Consequences
1. Reduces Athena scan volume compared with unpartitioned output.
2. Easier operational model than `year/quarter` over-partitioning for this dataset size.

---

## ADR-003: Schema Management Strategy

### Status
- Proposed

### Date
- 2026-02-12

### Context
Need stable and queryable schema in Glue for downstream Athena use.

### Options Considered
1. Glue crawler discovery.
2. Explicit table DDL managed in code.
3. Hybrid (crawler for exploration, DDL for production).

### Decision
- Selected option: Hybrid (crawler for exploration, DDL for productionized tables).

### Rationale
1. Fast schema discovery during early iteration.
2. Deterministic schema control once datasets stabilize.

### Consequences
1. Lower early friction with manageable schema drift risk.
2. Slight increase in maintenance once DDL governance is introduced.

---

## ADR-004: Orchestration and Scheduling

### Status
- Proposed

### Date
- 2026-02-12

### Context
Need dependable batch orchestration with clear operational visibility.

### Options Considered
1. Manual CLI runs only.
2. Step Functions + EventBridge Scheduler.
3. Airflow/MWAA.

### Decision
- Selected option: Step Functions + EventBridge Scheduler.

### Rationale
1. Native AWS integration with clear state visibility and failure handling.
2. Easy to operate daily schedules without running workflow infrastructure.

### Consequences
1. Lower operations burden than managing Airflow for this scope.
2. Clear and auditable success/failure paths for batch runs.

---

## Decision Summary Table

| ADR | Topic | Decision | Status | Last Reviewed |
|---|---|---|---|---|
| ADR-001 | EMR compute mode | Serverless default, EC2 benchmarked | Proposed | 2026-02-12 |
| ADR-002 | Curated data layout | Partition by `year` | Proposed | 2026-02-12 |
| ADR-003 | Schema management | Hybrid crawler plus DDL | Proposed | 2026-02-12 |
| ADR-004 | Orchestration | Step Functions plus EventBridge | Proposed | 2026-02-12 |

## Changelog

| Date | Change | Author |
|---|---|---|
| 2026-02-12 | Initial draft with proposed defaults | Data Engineering Owner (Weiming) |
