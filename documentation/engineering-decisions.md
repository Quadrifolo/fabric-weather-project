# Engineering Decisions

## Overview

This document records the principal engineering decisions made while designing the London Weather Data Platform in Microsoft Fabric.

The objective was not simply to visualise weather data, but to design an incremental data-processing architecture incorporating ingestion, validation, transformation, orchestration and persistent processing state.

---

## 1. External API as the Source

### Decision

Use the Open-Meteo Historical Weather API as the external data source.

### Rationale

API ingestion introduces engineering considerations that are not present when working only with static files, including:

- HTTP connectivity
- Query parameters
- JSON responses
- Schema validation
- Dynamic processing dates
- Retry behaviour
- Incremental ingestion

This made the project suitable for demonstrating an automated data-engineering workflow.

---

## 2. Preserve Raw API Responses in Bronze

### Decision

Store the original JSON response in the Bronze layer before transformation.

### Rationale

Transforming the API response immediately without retaining the source would reduce traceability.

Keeping raw responses provides:

- Source-level lineage
- Reprocessing capability
- Debugging support
- Separation between ingestion and transformation

The Bronze layer therefore acts as the immutable/raw landing area.

---

## 3. Medallion Architecture

### Decision

Use Bronze, Silver and Gold logical layers.

### Rationale

Each layer serves a distinct responsibility:

**Bronze**  
Raw API response.

**Silver**  
Structured and validated hourly observations.

**Gold**  
Daily analytical metrics.

This separation makes the processing flow easier to understand, test and maintain.

---

## 4. Hourly Grain for Silver

### Decision

Maintain individual hourly weather observations in the Silver layer.

### Rationale

Aggregating immediately to daily grain would remove useful detail.

Hourly Silver data preserves analytical flexibility while allowing Gold models to be rebuilt using different aggregation rules later.

---

## 5. Delta Lake for Structured Tables

### Decision

Store Silver and Gold outputs using Delta tables.

### Rationale

Delta provides a reliable structured storage layer and supports processing patterns such as `MERGE`, which is useful for incremental and retry-safe workflows.

---

## 6. Delta MERGE for Silver Processing

### Decision

Use merge-based processing rather than blindly appending every processed observation.

### Rationale

Incremental pipelines may need to retry a processing date following a failure.

A simple append strategy could introduce duplicate observations.

Merge-based processing provides a more idempotent design:

```text
Process date
    ↓
Observation already exists?
    ├── Yes → avoid duplicate insert
    └── No  → insert observation
```

This allows pipeline stages to be rerun more safely.

---

## 7. Separate Gold Transformation

### Decision

Create a dedicated Gold transformation stage that aggregates Silver observations to daily grain.

### Rationale

Power BI should consume business-ready analytical data rather than repeatedly transforming hourly observations.

The Gold table calculates:

- Average temperature
- Minimum temperature
- Maximum temperature
- Average humidity
- Total precipitation
- Average wind speed
- Maximum wind speed

This keeps the reporting layer lightweight.

---

## 8. Persistent Watermark Table

### Decision

Maintain incremental processing state in a Delta table rather than relying on manually supplied pipeline dates.

### Rationale

Manual `start_date` and `end_date` parameters require intervention for every ingestion cycle.

The watermark table provides persistent state:

```text
pipeline_name       last_successful_date
weather_ingestion   2026-08-03
```

The pipeline can derive:

```text
next_date = last_successful_date + 1 day
```

This converts the pipeline from manually driven ingestion into state-driven incremental processing.

---

## 9. Success-Gated Watermark Update

### Decision

Place watermark advancement at the end of the processing chain.

### Rationale

The watermark represents successfully processed data, not merely requested data.

If the watermark were updated immediately after API ingestion, a downstream Silver or Gold failure could incorrectly mark the date as completed.

The intended sequence is therefore:

```text
Get Watermark
      ↓
Ingest
      ↓
Validate
      ↓
Silver
      ↓
Gold
      ↓
Update Watermark
```

If processing fails before the final stage, the watermark remains unchanged and the date can be retried.

---

## 10. Date-Based Bronze File Naming

### Decision

Use processing dates in Bronze filenames.

Example:

```text
weather_2026-08-04.json
```

### Rationale

Date-based files provide:

- Clear lineage
- Easier troubleshooting
- Historical retention
- Easier reprocessing
- Protection against repeatedly overwriting one generic file

---

## 11. Separate Notebooks by Responsibility

### Decision

Separate major processing responsibilities into individual notebooks.

The project uses:

```text
00_setup_control_tables
01_bronze_validation
02_silver_transform
03_gold_transformation
04_update_watermark
```

### Rationale

This provides clear separation of concerns and allows individual stages to be developed and validated independently.

---

## 12. Gold as the Power BI Consumption Layer

### Decision

Connect reporting primarily to `gold_daily_weather`.

### Rationale

Business aggregation logic should be centralised upstream instead of duplicated inside report visuals.

This reduces Power BI complexity and creates a reusable analytical layer.

---

## 13. Capacity Constraint Handling

During development, the Microsoft Fabric trial environment encountered Spark capacity/concurrency limits when multiple Spark notebook activities were executed sequentially.

The error indicated that the available Spark capacity compute limit had been reached.

### Approach

Rather than redesigning the architecture solely around a temporary trial-capacity limitation:

- Individual notebook stages were tested independently.
- Pipeline dependencies were configured.
- Transformation outputs were validated.
- Incremental processing logic was implemented.
- The capacity limitation was documented transparently.

### Production Consideration

A production deployment would require appropriately sized Fabric capacity, concurrency monitoring and operational alerting.

This limitation relates to the development environment rather than the logical pipeline design.

---

## Summary

The resulting architecture prioritises:

- Traceability
- Separation of concerns
- Incremental processing
- Idempotency
- Retry safety
- Reusable analytical models
- Lightweight reporting

The project therefore demonstrates more than a simple API-to-dashboard workflow; it models several patterns commonly required in production data platforms.