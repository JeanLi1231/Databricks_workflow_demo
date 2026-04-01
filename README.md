# Databricks Workflow Demo: Basic DE + DS Pipeline

This project is a **demo** of a basic end-to-end Databricks data workflow.

It shows how to go from:
1. Medallion data preparation (ingest -> transform -> aggregate)
2. Workflow orchestration (configured as a Databricks Workflow job; notebooks in this repo run in that sequence)
3. Lightweight data analysis and prediction
4. Dashboard presentation of results

The analysis question is:

**"Does the time of day and pickup area predict how expensive a trip will be per mile?"**

## Scope

This repository is intentionally limited in scope.

- It is only meant to demonstrate a basic Databricks DE+DS workflow.
- It is not a thorough statistical analysis. The prediction model is a basic LinearRegression model with default parameters.


## Repository Structure

- `Notebooks/01_bronze.py.ipynb`: Ingest raw yellow taxi data and write Bronze Delta table.
- `Notebooks/02_silver.py.ipynb`: Apply data quality rules, derive features, deduplicate, and write Silver Delta table.
- `Notebooks/03_gold.py.ipynb`: Aggregate features by pickup location + hour and write Gold Delta table.
- `Notebooks/04_analysis.py.ipynb`: Train/evaluate simple regression model and write prediction output table.
- `Dashboards/taxt_price_prediction.lvdash.json`: Databricks Lakeview dashboard spec for visualizing predictions and errors.

## Data Pipeline (Medallion)

### 1) Bronze (Ingest)
Notebook: `01_bronze.py.ipynb`

- Reads source parquet files from:
  - `/Volumes/workspace/taxi_schema/taxi_raw_data/`
- Adds ingestion/audit metadata:
  - `_source_file`, `_file_modification_time`, `_ingested_at`
- Standardizes timestamp fields into:
  - `pickup_datetime`, `dropoff_datetime`
- Writes Delta table:
  - `workspace.taxi_schema.taxi_bronze`

### 2) Silver (Clean + Transform)
Notebook: `02_silver.py.ipynb`

- Reads from `workspace.taxi_schema.taxi_bronze`
- Filters invalid rows (null critical fields, invalid durations, negative values, out-of-range categories)
- Adds derived columns such as:
  - `trip_duration_minutes`, `pickup_hour`, `pickup_day_of_week`, `pickup_day_name`, `is_airport_trip`
- Builds business key and deduplicates by most recent `_ingested_at`
- Writes partitioned Delta table:
  - `workspace.taxi_schema.taxi_silver`
  - Partitioned by `pickup_year`, `pickup_month`

### 3) Gold (Aggregate)
Notebook: `03_gold.py.ipynb`

- Reads from `workspace.taxi_schema.taxi_silver`
- Computes `price_per_mile = fare_amount / trip_distance` (for valid positive fare/distance rows)
- Aggregates by `PULocationID` and `pickup_hour`
- Keeps groups with at least 30 trips
- Writes Delta table:
  - `workspace.taxi_schema.taxi_gold`

## Analysis + Prediction

Notebook: `04_analysis.py.ipynb`

- Reads from `workspace.taxi_schema.taxi_gold`
- Uses features:
  - `PULocationID`, `pickup_hour`
- Predicts target:
  - `avg_fare_per_mile`
- Trains `sklearn.linear_model.LinearRegression` with train/test split
- Reports metrics (`R^2`, `RMSE`, `MAE`)
- Retrains on full dataset for dashboard output
- Writes final table:
  - `workspace.taxi_schema.analysis_result`

`analysis_result` includes:
- `predicted_fare_per_mile`
- `actual_fare_per_mile`
- `prediction_error`
- `absolute_error`
- Supporting aggregated context (`trip_count`, `avg_trip_duration`, `avg_trip_distance`)

## Dashboard Output

Dashboard spec: `Dashboards/taxt_price_prediction.lvdash.json`

The dashboard visualizes the `analysis_result` table with:
- KPI counters (locations, avg prediction error, total trips)
- Average predicted fare by hour
- Actual vs predicted fare comparison
- Detail table by location and hour
- Heatmap of predicted price-per-mile by `PULocationID` and `pickup_hour`

## Suggested Databricks Workflow Order

Orchestrate these notebooks in Databricks Workflows in this sequence:

1. `Notebooks/01_bronze.py.ipynb`
2. `Notebooks/02_silver.py.ipynb`
3. `Notebooks/03_gold.py.ipynb`
4. `Notebooks/04_analysis.py.ipynb`

Then refresh/publish the Lakeview dashboard using `analysis_result` as the source.

## Prerequisites

Before running the notebooks, make sure:

- You are in a Databricks workspace with a Spark cluster/runtime available.
- The Unity Catalog objects exist (or you update names in notebooks):
  - Catalog: `workspace`
  - Schema: `taxi_schema`
- Raw parquet data is available at:
  - `/Volumes/workspace/taxi_schema/taxi_raw_data/`
- Python packages used in analysis notebook are available:
  - `scikit-learn`, `pandas`, `numpy`