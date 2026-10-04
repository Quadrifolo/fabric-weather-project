# Lessons Learned

## Overview

The London Weather Data Platform introduced several engineering concepts beyond the static batch-processing architecture used in the NYC Green Taxi project.

The most significant learning areas were API ingestion, pipeline orchestration, Delta Lake processing, incremental loading and persistent processing state.

---

## 1. API Ingestion Requires More Than Retrieving Data

Connecting to an API is only the first stage of an ingestion solution.

A reliable workflow must also consider:

- API parameters
- Response structure
- Required fields
- Storage of the original response
- Schema changes
- Retry behaviour
- Incremental processing

Retaining the original JSON response in Bronze proved useful because ingestion and transformation could be treated as separate concerns.

---

## 2. Raw Data Should Be Preserved

Keeping the original API response provides an important recovery point.

If transformation logic changes, Silver and Gold can be rebuilt from Bronze without requiring the source API to be called again.

This improves traceability and reproducibility.

---

## 3. Schema Validation Should Happen Early

External APIs can change independently of downstream processing logic.

Validating required fields in Bronze creates an early failure point when the expected data contract is no longer satisfied.

It is preferable for a pipeline to fail clearly during validation rather than produce incomplete analytical data.

---

## 4. Delta Schema Management Matters

During development, a schema mismatch occurred when writing to an existing Delta table.

Resolving the issue required explicit schema-evolution handling using `mergeSchema`.

This demonstrated that Delta tables provide structure and reliability, but schema changes still need to be managed deliberately.

---

## 5. Idempotency Is Important for Incremental Pipelines

A pipeline that may retry a processing date should not simply append every observation again.

Using Delta `MERGE` provides a safer pattern because existing observations can be identified before inserting new records.

This becomes particularly important when orchestration failures occur after ingestion but before the entire pipeline completes.

---

## 6. Incremental Processing Requires Persistent State

Initial pipeline versions relied on manually supplied start and end dates.

Introducing the `pipeline_watermark` table changed the architecture from manually controlled ingestion to state-driven processing.

Instead of asking:

> What date should this pipeline process?

the pipeline can determine:

```text
last successful date + 1 day
```

This is a fundamental pattern for incremental data engineering.

---

## 7. A Watermark Must Represent Successful Processing

The watermark should not be updated simply because data was requested from the source.

It should represent the latest date that successfully completed the required processing workflow.

Placing the watermark update after downstream transformations provides safer retry behaviour.

---

## 8. Separate Responsibilities Make Pipelines Easier to Reason About

Using separate notebooks for:

- Control-table setup
- Bronze validation
- Silver transformation
- Gold transformation
- Watermark updates

made individual stages easier to understand and test.

This separation was especially useful when capacity limitations prevented reliable execution of the entire pipeline at once.

---

## 9. Fabric Capacity Is Part of Pipeline Design

The project encountered Spark capacity limits within the Microsoft Fabric trial environment.

Multiple sequential notebook activities can require Spark/Livy sessions, and available compute capacity can therefore influence orchestration behaviour.

This highlighted that production data engineering involves both transformation logic and platform-resource considerations.

A logically correct pipeline can still fail when insufficient compute resources are available.

---

## 10. Environmental Failures Should Be Distinguished From Code Failures

The capacity errors initially appeared as failed notebook activities.

Reviewing the underlying errors showed that the failures occurred while creating Spark sessions rather than while executing transformation logic.

This reinforced the importance of reading platform run logs and distinguishing:

- Code errors
- Data-quality failures
- Configuration issues
- Resource/capacity failures

These require different responses.

---

## 11. Gold Tables Simplify Reporting

Creating `gold_daily_weather` upstream allowed Power BI to remain relatively simple.

The report did not need to repeatedly calculate daily temperature, precipitation, humidity and wind aggregations from raw hourly observations.

This reinforced the principle that reusable business logic generally belongs in the analytical data model rather than being recreated separately in every visual.

---

## 12. Aggregation Semantics Matter in Power BI

Power BI initially defaulted some metrics to `Sum`.

This produced misleading results when fields such as daily average temperature and humidity were added together.

The correct aggregation depends on the meaning of the metric:

```text
Average temperature → Average
Maximum temperature → Maximum
Total precipitation → Sum
Average humidity → Average
Average wind speed → Average
```

A technically correct Gold model can still produce misleading reporting if semantic aggregation is configured incorrectly.

---

## 13. The Medallion Pattern Works Well Beyond Static Files

The NYC Taxi project applied Bronze, Silver and Gold to a static Parquet dataset.

The Weather project demonstrated that the same conceptual architecture can be extended to API-driven ingestion:

```text
API
 ↓
Bronze JSON
 ↓
Silver hourly observations
 ↓
Gold daily metrics
 ↓
Power BI
```

The architecture is therefore a reusable design pattern rather than something specific to one source format.

---

## Overall Takeaway

The most important progression in this project was moving from:

```text
Load data
→ Transform data
→ Visualise data
```

towards:

```text
Maintain processing state
        ↓
Ingest external data
        ↓
Preserve raw source
        ↓
Validate
        ↓
Transform safely
        ↓
Create analytical models
        ↓
Advance processing state
        ↓
Report
```

That progression introduced several concepts required in real data platforms: orchestration, state, idempotency, retries, schema management, validation and platform-resource constraints.

These concepts provide the foundation for more advanced work involving scheduling, monitoring, CI/CD, multi-environment deployment and larger-scale incremental processing.