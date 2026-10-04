# London Weather Data Platform

An end-to-end Microsoft Fabric data engineering project that ingests historical London weather data from the Open-Meteo API and processes it through an incremental Bronze, Silver and Gold architecture for analytical reporting in Power BI.

The project demonstrates API ingestion, Microsoft Fabric Data Pipelines, PySpark, Delta Lake, data-quality validation, incremental processing, watermark-based state management and Power BI reporting.

---

## Project Overview

The objective of this project was to build an automated weather data platform rather than simply analyse a static dataset.

Historical weather observations are retrieved from the Open-Meteo API through a Microsoft Fabric Data Pipeline and stored as raw JSON in a Fabric Lakehouse.

The data is then:

1. Preserved in the Bronze layer.
2. Validated against the expected API structure.
3. Transformed into structured hourly Silver observations.
4. Processed using Delta Lake to support safe repeated execution.
5. Aggregated into daily Gold weather metrics.
6. Exposed to Power BI for analytical reporting.

A persistent watermark table tracks the latest successfully processed date and supports incremental ingestion.

---

# Architecture

![Weather Platform Architecture](images/Architecture.png)

```text
Open-Meteo Historical Weather API
                │
                ▼
        Fabric Data Pipeline
                │
                ▼
        Get Current Watermark
                │
                ▼
         Copy Weather API
                │
                ▼
        Bronze JSON / OneLake
                │
                ▼
       Bronze Layer Validation
                │
                ▼
     Silver Weather Observations
        Delta Lake + MERGE
                │
                ▼
        Gold Daily Weather
                │
                ▼
              Power BI
                │
                ▼
       Update Pipeline Watermark
```

The watermark is advanced only after the required downstream processing has completed successfully.

---

## Technology Stack

- Microsoft Fabric
- Fabric Data Factory / Data Pipelines
- Fabric Lakehouse
- OneLake
- Apache Spark
- PySpark
- Delta Lake
- REST / HTTP APIs
- JSON
- Power BI
- Git / GitHub

---

# Pipeline Orchestration

The ingestion and transformation process is orchestrated through Microsoft Fabric Data Pipelines.

![Fabric Weather Pipeline](pipeline/pipeline.png)

The pipeline follows the logical sequence:

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

Each transformation stage depends on successful completion of the required upstream stage.

The final watermark activity is success-gated so that a failed transformation does not intentionally advance processing state.

---

# Medallion Architecture

## Bronze — Raw API Data

Weather observations are retrieved from the Open-Meteo API and stored as raw JSON within the Lakehouse Files area.

Example:

```text
Files/
└── bronze/
    └── weather/
        └── raw/
            └── weather_YYYY-MM-DD.json
```

The Bronze layer preserves the original API response before analytical transformations are applied.

### Purpose

- Preserve source-level data
- Maintain traceability
- Support reprocessing
- Separate ingestion from transformation
- Provide a debugging/recovery point

Before downstream processing, the Bronze notebook validates that required hourly fields are present.

Required fields include:

```text
time
temperature_2m
relative_humidity_2m
precipitation
wind_speed_10m
```

---

## Silver — Structured Hourly Observations

Raw JSON is transformed into a structured Delta table:

`silver_weather_observations`

The Silver layer represents weather data at **hourly observation grain**.

Processing includes:

- Extraction of nested hourly API values
- Timestamp standardisation
- Data-type handling
- Column standardisation
- Required-field validation
- Delta table writes
- Duplicate-safe incremental processing

Keeping hourly observations in Silver preserves analytical flexibility while providing a trusted structured dataset for downstream modelling.

---

# Incremental Processing & Delta MERGE

A key objective of the project was to move beyond simple append-based ingestion.

Incremental pipelines may need to retry a processing date after a downstream failure. Blindly appending the same source observations again could therefore create duplicates.

The Silver processing layer uses Delta Lake `MERGE` logic to support safer repeated execution.

Conceptually:

```text
Incoming observation
        │
        ▼
Does observation already exist?
        │
     ┌──┴──┐
    Yes    No
     │      │
     ▼      ▼
 No duplicate   Insert
   insert       record
```

This makes the processing workflow more idempotent and retry-friendly.

---

# Watermark-Based Ingestion

Incremental processing state is maintained in:

`pipeline_watermark`

The table stores:

```text
pipeline_name
last_successful_date
```

Example:

```text
weather_ingestion | 2026-08-03
```

The pipeline retrieves this value before ingestion and derives the next processing date.

```text
Last successful date
2026-08-03
      ↓
+ 1 day
      ↓
Next processing date
2026-08-04
```

Following successful downstream processing:

```text
2026-08-03
      ↓
2026-08-04
```

If processing fails before the watermark-update stage, the stored date remains unchanged so that the same processing period can be retried.

This removes the need to manually supply pipeline `start_date` and `end_date` parameters for each incremental run.

---

# Gold — Daily Weather Analytics

Silver observations are aggregated into:

`gold_daily_weather`

The Gold table contains one row per observation date and provides:

- Average temperature
- Minimum temperature
- Maximum temperature
- Average relative humidity
- Total precipitation
- Average wind speed
- Maximum wind speed

Conceptually:

```text
Hourly Silver Observations
          ↓
Group by Observation Date
          ↓
Daily Weather Metrics
          ↓
gold_daily_weather
```

Creating these aggregations upstream provides Power BI with an analytics-ready dataset and reduces transformation requirements in the reporting layer.

---

# Power BI Dashboard

![London Weather Overview](images/Overview.png)

The final Power BI report provides an analytical overview of historical London weather.

## KPIs

- Average Temperature
- Maximum Temperature
- Total Precipitation
- Average Humidity

## Trends

- Daily Temperature
- Average Humidity
- Daily Precipitation
- Average Wind Speed

Metric aggregation was configured according to the meaning of each field.

For example:

| Metric | Aggregation |
|---|---|
| Average Temperature | Average |
| Maximum Temperature | Maximum |
| Total Precipitation | Sum |
| Average Humidity | Average |
| Average Wind Speed | Average |

Power BI primarily consumes the Gold layer rather than reproducing transformations from raw hourly data.

---

# Data Quality

Validation is performed at multiple stages.

## Bronze

- Validate API response structure
- Validate required hourly fields
- Confirm source data availability

## Silver

- Validate timestamps
- Validate expected schema
- Validate numeric data types
- Protect against duplicate processing
- Validate successful Delta writes

## Gold

- Validate daily grain
- Validate temperature aggregations
- Validate humidity aggregation
- Validate precipitation totals
- Validate wind-speed aggregations

## Reporting

Power BI aggregations are reconciled with the corresponding Gold metrics.

Further detail is available in:

[`documentation/data-quality-report.md`](documentation/data-quality-report.md)

and:

[`documentation/testing.md`](documentation/testing.md)

---

# Notebook Structure

Processing responsibilities are separated across five Fabric notebooks.

```text
notebooks/
├── 00_setup_control_tables.ipynb
├── 01_bronze_validation.ipynb
├── 02_silver_transform.ipynb
├── 03_gold_transformation.ipynb
└── 04_update_watermark.ipynb
```

### `00_setup_control_tables`

Creates the persistent control structures used for incremental processing, including the pipeline watermark table.

### `01_bronze_validation`

Loads the raw API response and validates that the required hourly fields are available.

### `02_silver_transform`

Transforms raw weather observations into the structured Silver Delta model and implements incremental-safe processing.

### `03_gold_transformation`

Aggregates hourly Silver observations into daily analytical weather metrics.

### `04_update_watermark`

Advances the processing watermark following successful downstream processing.

---

# Testing

Testing was performed throughout the workflow.

| Component | Validation |
|---|---|
| API ingestion | API response successfully retrieved |
| Bronze storage | Raw JSON successfully persisted |
| Bronze validation | Required hourly fields validated |
| Silver transformation | Structured hourly observations created |
| Delta processing | Schema handling and MERGE logic implemented |
| Gold transformation | Daily analytical metrics generated |
| Watermark | Persistent state created and retrieved |
| Pipeline | Dependencies and success paths configured |
| Power BI | Gold metrics consumed successfully |

Detailed testing information is available in:

[`documentation/testing.md`](documentation/testing.md)

---

# Development Constraint — Fabric Trial Capacity

During development, full sequential execution of the pipeline encountered Microsoft Fabric trial Spark capacity limits.

Fabric returned:

```text
TooManyRequestsForCapacity
HTTP 430
```

when additional Spark/Livy sessions could not be created within the available trial compute capacity.

This was treated as an environment constraint rather than hidden from the project documentation.

To validate the implementation:

- Bronze ingestion was tested successfully.
- Bronze validation was tested successfully.
- Silver transformation was tested independently.
- Gold transformation was tested independently.
- Delta outputs were inspected.
- Watermark persistence was tested.
- Pipeline dependencies were configured and validated.
- Power BI successfully consumed the resulting Gold layer.

A production implementation would require appropriately sized Fabric capacity together with operational monitoring and alerting.

---

# Engineering Decisions

Important design decisions included:

- Preserve original API responses before transformation
- Separate Bronze, Silver and Gold responsibilities
- Maintain hourly grain in Silver
- Use Delta tables for structured processing
- Use `MERGE` for safer repeated Silver processing
- Maintain persistent incremental state through a watermark
- Advance the watermark only after successful processing
- Use date-based Bronze files for lineage
- Separate major processing stages into individual notebooks
- Push daily aggregation upstream into Gold rather than Power BI

Detailed rationale is available in:

[`documentation/engineering-decisions.md`](documentation/engineering-decisions.md)

---

# Lessons Learned

Key lessons from the project included:

- API ingestion requires validation and state management, not just connectivity.
- Raw source data should be preserved for traceability and reprocessing.
- Schema validation should occur before downstream transformation.
- Delta schema evolution needs to be managed explicitly.
- Incremental pipelines benefit from idempotent processing.
- Persistent watermarks provide a clean mechanism for state-driven ingestion.
- Watermarks should represent successfully processed data rather than simply requested data.
- Platform capacity and Spark concurrency can affect otherwise valid pipeline designs.
- Gold analytical models significantly simplify downstream reporting.
- Correct semantic aggregation in Power BI remains important even when upstream models are well designed.

See:

[`documentation/lessons-learned.md`](documentation/lessons-learned.md)

---

# Repository Structure

```text
fabric-weather-data-platform/
│
├── architecture/
│   └── weather-platform-architecture.png
│
├── documentation/
│   ├── business-requirements.md
│   ├── data-dictionary.md
│   ├── data-quality-report.md
│   ├── engineering-decisions.md
│   ├── lessons-learned.md
│   └── testing.md
│
├── images/
│   ├── lakehouse-overview.png
│   ├── bronze-weather-files.png
│   ├── silver-weather-observations.png
│   ├── gold-daily-weather.png
│   └── pipeline-watermark.png
│
├── notebooks/
│   ├── 00_setup_control_tables.ipynb
│   ├── 01_bronze_validation.ipynb
│   ├── 02_silver_transform.ipynb
│   ├── 03_gold_transformation.ipynb
│   └── 04_update_watermark.ipynb
│
├── pipeline/
│   ├── README.md
│   └── screenshots/
│       └── weather-ingestion-pipeline.png
│
├── power-bi/
│   └── screenshots/
│       └── london-weather-overview.png
│
├── sample-output/
│
├── .gitignore
├── LICENSE
└── README.md
```

---

# How to Reproduce

1. Create a Microsoft Fabric workspace.
2. Create a Fabric Lakehouse.
3. Attach the Lakehouse to the project notebooks.
4. Configure an HTTP connection for the Open-Meteo API.
5. Run `00_setup_control_tables.ipynb` to create the watermark structure.
6. Create the Fabric Data Pipeline.
7. Configure the watermark lookup.
8. Configure the API Copy activity to land raw JSON in Bronze.
9. Configure the notebook activities in processing order.
10. Run Bronze validation.
11. Transform observations into the Silver Delta table.
12. Generate the Gold daily weather table.
13. Configure the watermark update as a success-dependent final activity.
14. Connect `gold_daily_weather` to Power BI.
15. Build or reproduce the analytical dashboard.

---

# Documentation

Additional project documentation is available in the `documentation/` directory:

- [Business Requirements](documentation/business-requirements.md)
- [Data Dictionary](documentation/data-dictionary.md)
- [Data Quality Report](documentation/data-quality-report.md)
- [Engineering Decisions](documentation/engineering-decisions.md)
- [Testing](documentation/testing.md)
- [Lessons Learned](documentation/lessons-learned.md)

---

# Project Progression

This project builds on the batch-oriented NYC Green Taxi Lakehouse project by introducing additional data-engineering concepts including:

- External REST API ingestion
- Fabric Data Pipeline orchestration
- Incremental processing
- Delta `MERGE`
- Persistent watermarking
- Retry-safe processing design
- Pipeline state management

Together, the two projects demonstrate progression from batch Lakehouse analytics toward a more automated and stateful data-engineering architecture.