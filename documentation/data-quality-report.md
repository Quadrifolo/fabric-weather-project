# Data Quality Report

## Overview

Data-quality validation is applied throughout the London Weather Data Platform to prevent malformed or unsuitable source data from silently propagating into downstream analytical datasets.

Validation occurs across the Bronze, Silver and Gold layers.

---

## Bronze Validation

The Bronze layer retains the original Open-Meteo API response.

Before transformation, the structure of the API response is inspected to confirm that the required hourly fields are available.

Required fields:

- `time`
- `temperature_2m`
- `relative_humidity_2m`
- `precipitation`
- `wind_speed_10m`

The validation logic compares the expected field list against the schema contained within the `hourly` section of the API response.

If required fields are missing, processing raises an exception rather than continuing with an incomplete dataset.

Example validation outcome:

```text
Bronze hourly structure validation passed.
```

This provides an early schema-contract check between the external API and the transformation layer.

---

## Silver Validation

The Silver transformation converts nested API data into structured hourly observations.

Validation considerations include:

### Schema

The expected analytical fields must exist following transformation.

### Timestamp Integrity

API timestamps are converted into an appropriate timestamp representation before downstream aggregation.

### Data Types

Weather measurements must use appropriate numeric types rather than remaining as unstructured strings.

### Required Values

Fields required for analytical calculations are checked during processing.

### Duplicate Processing

The Silver layer uses Delta Lake processing designed to support repeated pipeline execution without blindly appending duplicate observations.

This is particularly important because failed incremental pipeline runs may need to process the same date again.

---

## Gold Validation

The Gold layer aggregates hourly Silver observations to daily grain.

The following calculations are validated conceptually against their Silver inputs:

- Average temperature
- Minimum temperature
- Maximum temperature
- Average humidity
- Total precipitation
- Average wind speed
- Maximum wind speed

The expected Gold grain is:

**one row per observation date**

This provides a straightforward validation rule for detecting unexpected duplication in the reporting layer.

---

## Pipeline State Validation

Incremental processing introduces an additional quality requirement: processing state must remain consistent with successfully processed data.

The `pipeline_watermark` table records the latest successfully processed date.

The watermark design follows two principles:

1. The next ingestion date is derived from the current watermark.
2. The watermark should only advance after successful downstream processing.

This prevents failed runs from incorrectly marking unprocessed data as complete.

---

## Power BI Validation

Power BI consumes the Gold layer rather than rebuilding the transformation logic from Bronze or Silver.

Report metrics include:

- Average temperature
- Maximum temperature
- Total precipitation
- Average humidity
- Temperature trends
- Humidity trends
- Precipitation trends
- Wind-speed trends

Aggregations were configured according to metric meaning.

For example:

- Average temperature → Average
- Maximum temperature → Maximum
- Total precipitation → Sum
- Average humidity → Average
- Average wind speed → Average

This prevents inappropriate default aggregations such as summing daily average temperatures.

---

## Known Data Quality Considerations

The project uses an external API and therefore depends on the availability and structure of upstream source data.

Potential production considerations include:

- Missing API observations
- API schema changes
- Unexpected null values
- Extreme weather values
- Duplicate API responses
- Partial API responses
- Changes to measurement units

A production implementation could extend validation with formal acceptable ranges, anomaly detection and automated alerting.

---

## Outcome

The implemented validation strategy provides checks across:

```text
Source structure
      ↓
Bronze schema validation
      ↓
Silver structural validation
      ↓
Duplicate-safe processing
      ↓
Gold grain/aggregation validation
      ↓
Power BI reconciliation
```

This provides a stronger analytical foundation than relying solely on successful pipeline execution as evidence of data quality.