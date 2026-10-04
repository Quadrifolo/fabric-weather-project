# Business Requirements

## Project Overview

The London Weather Data Platform is an end-to-end Microsoft Fabric data engineering solution designed to ingest historical weather observations from the Open-Meteo API, process the data through a Medallion Architecture, and expose business-ready daily weather metrics for analytical reporting in Power BI.

The project extends beyond a traditional batch analytics solution by introducing API-based ingestion, pipeline orchestration, Delta Lake processing, data-quality validation, incremental loading and persistent watermark management.

## Business Objective

The objective of the solution is to provide a reliable and reusable data pipeline capable of collecting, processing and analysing historical London weather observations.

The platform should:

- Retrieve weather observations from an external REST API.
- Preserve raw API responses for traceability and reprocessing.
- Validate the structure of incoming weather data.
- Transform raw observations into a consistent analytical schema.
- Prevent duplicate observations during repeated or incremental processing.
- Produce daily weather metrics suitable for reporting.
- Track the latest successfully processed date.
- Support incremental ingestion without requiring manual date entry.
- Expose curated weather data to Power BI.

## Data Source

Weather data is sourced from the Open-Meteo Historical Weather API.

The ingestion process retrieves hourly observations including:

- Temperature
- Relative humidity
- Precipitation
- Wind speed
- Observation timestamp

The API response is stored in its original JSON format within the Bronze layer before downstream transformation.

## Functional Requirements

### 1. API Data Ingestion

The solution must retrieve historical weather observations from the Open-Meteo API using Microsoft Fabric Data Pipelines.

The ingestion process should:

- Call the external API through an HTTP connection.
- Request the required hourly weather variables.
- Determine the date to process using pipeline state rather than requiring manual date input.
- Store each API response in the Lakehouse for downstream processing.

### 2. Bronze Layer

Raw API responses must be retained within the Lakehouse Files area.

The Bronze layer should:

- Preserve the original JSON response.
- Maintain source-level traceability.
- Allow data to be reprocessed if transformation logic changes.
- Separate ingestion from downstream transformation.

Raw weather files should be organised under a dedicated Bronze weather directory.

### 3. Bronze Data Validation

Before transformation, the incoming API structure must be validated.

The pipeline should verify that the expected hourly weather fields are present, including:

- `time`
- `temperature_2m`
- `relative_humidity_2m`
- `precipitation`
- `wind_speed_10m`

Processing should fail when mandatory fields required by downstream transformations are missing.

### 4. Silver Layer

Raw weather observations must be transformed into a structured trip-independent analytical dataset at hourly observation grain.

The Silver layer should:

- Extract hourly observations from the nested API response.
- Standardise column names.
- Apply appropriate data types.
- Convert observation timestamps into a consistent timestamp representation.
- Validate required values.
- Produce one structured record per hourly weather observation.
- Support safe repeated execution.

The resulting Silver Delta table is:

`silver_weather_observations`

### 5. Incremental and Idempotent Processing

The pipeline should support incremental processing without introducing duplicate Silver records.

Delta Lake `MERGE` logic should be used to ensure that an observation already present in the Silver table is not blindly appended again during a retry or repeated pipeline execution.

This allows the processing workflow to be safely rerun where required.

### 6. Gold Layer

The Silver dataset must be transformed into a daily analytical model suitable for reporting.

The Gold layer should calculate:

- Average daily temperature
- Minimum daily temperature
- Maximum daily temperature
- Average daily relative humidity
- Total daily precipitation
- Average daily wind speed
- Maximum daily wind speed

The resulting Gold Delta table is:

`gold_daily_weather`

The Gold layer should contain one record per observation date.

## Analytical Requirements

The final analytical layer should support questions such as:

- What was the average temperature for each day?
- What were the minimum and maximum daily temperatures?
- How did temperature change over time?
- How did average humidity change over time?
- How much precipitation occurred each day?
- How did average wind speed change over time?
- What was the maximum wind speed recorded during each day?

## Pipeline Orchestration Requirements

Microsoft Fabric Data Pipelines should orchestrate the end-to-end workflow.

The logical pipeline sequence is:

```text
Get Watermark
      ↓
Copy Weather API
      ↓
Bronze Layer Validation
      ↓
Silver Transformation
      ↓
Gold Transformation
      ↓
Update Watermark
```

Each downstream transformation should depend on successful completion of the required upstream stage.

The watermark must only advance after successful processing of the corresponding data.

## Watermark Requirements

The platform should maintain persistent pipeline state using a Delta control table.

The control table:

`pipeline_watermark`

stores:

- Pipeline identifier
- Last successfully processed date

Before ingestion, the pipeline should retrieve the current watermark and determine the next date requiring processing.

For example:

```text
Last successful date: 2026-08-03
Next processing date:  2026-08-04
```

Following successful processing, the watermark should advance:

```text
2026-08-03
     ↓
2026-08-04
```

If downstream processing fails, the watermark should remain unchanged so the same processing date can be retried.

## Reporting Requirements

The curated Gold layer should be consumable by Power BI.

The final report should provide an overview of London weather conditions including:

- Average temperature
- Maximum temperature
- Total precipitation
- Average humidity
- Daily temperature trend
- Daily humidity trend
- Daily precipitation
- Daily wind-speed trend

Power BI should primarily consume the Gold layer rather than reproducing transformation logic from the raw API response.

## Data Quality Requirements

The solution should include validation at multiple stages.

### Bronze

Validate:

- Required API structure
- Required hourly fields
- Availability of source data

### Silver

Validate:

- Data types
- Observation timestamps
- Required values
- Duplicate observations
- Successful Delta writes

### Gold

Validate:

- Daily aggregation grain
- Temperature aggregations
- Humidity aggregations
- Precipitation totals
- Wind-speed aggregations

### Reporting

Power BI metrics should be reconciled against the corresponding Gold data.

## Non-Functional Requirements

The solution should be:

**Traceable**  
Raw API responses should remain available in Bronze.

**Reproducible**  
Silver and Gold datasets should be rebuildable from upstream data.

**Idempotent**  
Repeated processing should not introduce duplicate observations.

**Incremental**  
The pipeline should process new dates using persistent pipeline state.

**Modular**  
Ingestion, validation, Silver transformation, Gold transformation and watermark management should remain logically separated.

**Analytics-ready**  
The final Gold model should minimise transformation requirements within Power BI.

## Success Criteria

The project is considered successful when:

- Weather data can be retrieved from the Open-Meteo API.
- Raw JSON responses are retained in Bronze.
- Incoming API structure is validated.
- Hourly observations are transformed into a structured Silver Delta table.
- Reprocessing does not blindly duplicate existing observations.
- Daily analytical metrics are generated in the Gold layer.
- Pipeline state is maintained through a persistent watermark.
- The orchestration design advances the watermark only following successful processing.
- Gold weather data can be consumed and visualised in Power BI.

## Scope

This project focuses on demonstrating an incremental API-driven data engineering architecture in Microsoft Fabric.

The primary focus is the engineering workflow:

**API ingestion → Bronze → validation → Silver → Gold → incremental state management → Power BI**

Advanced production capabilities such as enterprise monitoring, alerting, CI/CD, multiple environments and large-scale streaming ingestion are outside the current project scope.