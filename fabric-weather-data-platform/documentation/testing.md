# Testing

## Overview

Testing was performed throughout the London Weather Data Platform to validate ingestion, transformation, data quality, analytical outputs and incremental-processing behaviour.

Because the project was developed using Microsoft Fabric trial capacity, individual stages were also tested independently where Spark concurrency limits prevented reliable execution of the entire pipeline in a single run.

---

## 1. API Ingestion Testing

### Objective

Confirm that Microsoft Fabric can successfully retrieve historical weather data from the Open-Meteo API.

### Validation

- HTTP connection successfully established.
- API returned a valid JSON response.
- Requested hourly weather fields were present.
- Response successfully landed in the Lakehouse Files area.

### Result

**Passed**

---

## 2. Bronze Storage Testing

### Objective

Confirm that raw API responses are retained before transformation.

### Validation

- JSON response written to the Bronze weather directory.
- Source file accessible from the Lakehouse.
- Raw API structure preserved.

### Result

**Passed**

---

## 3. Bronze Schema Validation

### Objective

Prevent downstream processing when required API fields are unavailable.

### Required Fields

```text
time
temperature_2m
relative_humidity_2m
precipitation
wind_speed_10m
```

### Validation Logic

The Bronze notebook compares the required fields against the fields contained within the API response's hourly schema.

### Result

**Passed**

The validation returned:

```text
Bronze hourly structure validation passed.
```

---

## 4. Silver Transformation Testing

### Objective

Confirm that nested Bronze JSON can be converted into structured hourly weather observations.

### Validation

- Hourly arrays successfully transformed.
- Observation timestamps created correctly.
- Required weather metrics retained.
- Output successfully written as a Delta table.
- Table accessible as `silver_weather_observations`.

### Result

**Passed**

---

## 5. Schema Evolution Testing

During development, a Delta schema mismatch was encountered while writing transformed data.

The write operation was updated to support the required schema evolution using:

```python
.option("mergeSchema", "true")
```

### Result

The updated write completed successfully.

This issue highlighted the importance of explicit schema management when evolving Delta tables.

---

## 6. Duplicate / Retry-Safety Testing

### Objective

Ensure repeated processing does not blindly append duplicate hourly observations.

### Approach

Silver processing uses Delta `MERGE` logic based on the observation identifier/timestamp.

### Expected Behaviour

```text
Existing observation → no duplicate insert
New observation      → insert
```

### Result

Merge-based processing was implemented to support idempotent/retry-safe execution.

---

## 7. Gold Transformation Testing

### Objective

Confirm that hourly Silver observations can be aggregated into daily analytical metrics.

### Validation

The Gold transformation generated:

- Average temperature
- Minimum temperature
- Maximum temperature
- Average humidity
- Total precipitation
- Average wind speed
- Maximum wind speed

Output table:

`gold_daily_weather`

Expected grain:

**one row per observation date**

### Result

**Passed**

---

## 8. Watermark Table Testing

### Objective

Confirm that persistent incremental-processing state can be stored and retrieved.

Initial test state:

```text
pipeline_name       last_successful_date
weather_ingestion   2026-08-03
```

### Validation

- Control table successfully created.
- Watermark successfully retrieved.
- Next processing date could be calculated from the stored value.

### Result

**Passed**

---

## 9. Pipeline Dependency Testing

### Objective

Ensure processing follows the required logical sequence.

Configured flow:

```text
Get Watermark
      ↓
Copy Weather API
      ↓
Bronze Layer Validation
      ↓
02_silver_transform
      ↓
03_gold_transformation
      ↓
04_update_watermark
```

The watermark activity is configured as a downstream success dependency so that processing state is not intentionally advanced before transformation completes.

---

## 10. Power BI Validation

### Objective

Confirm that Gold metrics are represented correctly in the reporting layer.

### Aggregation Checks

| Metric | Power BI Aggregation |
|---|---|
| Average Temperature | Average |
| Maximum Temperature | Maximum |
| Total Precipitation | Sum |
| Average Humidity | Average |
| Average Wind Speed | Average |

The precipitation visual was configured to display actual precipitation values rather than percentage composition.

### Result

**Passed**

---

## 11. End-to-End Pipeline Capacity Test

During full pipeline execution, the Fabric trial environment encountered:

```text
TooManyRequestsForCapacity
HTTP 430
```

The error occurred while Fabric attempted to create additional Spark/Livy sessions for sequential notebook activities.

### Impact

The available trial capacity prevented reliable full-chain execution despite individual processing stages functioning successfully.

### Validation Approach

To distinguish platform capacity from transformation defects:

- Bronze processing was executed independently.
- Silver processing was executed independently.
- Gold processing was executed independently.
- Watermark logic was implemented separately.
- Pipeline dependencies were inspected and configured.
- Produced datasets were validated directly.

### Outcome

Individual components were validated successfully.

The full orchestration limitation is documented as an environmental constraint and should not be interpreted as evidence of successful production-scale orchestration.

---

## Test Summary

| Component | Status |
|---|---|
| API ingestion | Passed |
| Bronze storage | Passed |
| Bronze schema validation | Passed |
| Silver transformation | Passed |
| Delta write/schema handling | Passed |
| Merge-based processing | Implemented |
| Gold transformation | Passed |
| Watermark persistence | Passed |
| Pipeline dependency configuration | Passed |
| Power BI consumption | Passed |
| Full sequential orchestration | Constrained by trial capacity |

---

## Future Testing Improvements

A production implementation should additionally include:

- Automated unit/data-quality tests
- Formal acceptable-value ranges
- Pipeline failure alerts
- Automated reconciliation
- Retry testing
- Missing-data scenarios
- API failure simulation
- Schema-change simulation
- Larger historical backfills
- Performance/load testing