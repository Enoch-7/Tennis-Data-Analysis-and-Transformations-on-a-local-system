# Tennis_Betting_Data_Pipeline_Architecture
End to end data transformation pipeline on a local system. Designed as a multi-layer architecture (source systems --> dlt ingestion -> storage system --> dbt transformation --> ML workloads), everything runs inside a reproducible Python environment. 

## Layers
**Environment**: uv, pyproject.toml with all dependencies
**Ingestion** : dlt : load matches, players and rankings into the DuckDB raw schema.
**Storage** : DuckDB + Parquet for raw archive
**Transform** : dbt-DuckDB
**Compute** : ELO(Polars) models/elo.py computes overall and per-surface Elo.
**BI** : Streamlit & Plotly 
**Orchestration** : Dagster (two assets : ...)

## Source and Ingestion
Jeff Sackmann : https://github.com/JeffSackmann/tennis_MatchChartingProject
Tennis Data App : https://tennisdata.app/downloads/
tennis-data.co : https://www.tennis-data.co.uk/data.php (Odds data)

**Ingestion techniques** : 
- Manual drop to data/inbox/ : run_inbox.py
- Scheduled dlt script : dlt_ingest.py

Any asset that lands in the destination folder activates a Dagster sensor. After loading, files move to data/archive/<date>/, so every ingestion is reproducible and auditable.    


## Data Modeling
Medallion layering inside one DuckDB file (data/tennis.duckdb), with schemas raw --> staging -->intermediate --> marts.

**Dimensional model (Kimball-style star schema)** : 
DIMENSIONS : player, tournament, date, surface
FACTS : match context, ranking snapshot, ELO rating, odds, predictions

**Marts for BI** :
Monthly ELO rankings, ELO vs ATP gap, Upcoming predictions, Modelo performance

## Orchestration with Dasgster
dlt assets (raw) ─▶ dbt staging/intermediate ─▶ elo_ratings (Polars asset) ─▶ dbt marts ─▶ predictions asset

- dagster-dlt and dagster-dbt expose each as assets automatically, so we have lineage in the UI.
- Schedules: weekly ingestion during the season and a monthly full refresh that triggers the dashboard rebuild.
- Sensor: watches data/inbox/ and triggers the manual-ingestion job when a new file appears.
- Asset checks: row-count and freshness checks that fail loudly before bad data reaches the dashboards.

## Project structure
tennis-engine/
├── pyproject.toml / uv.lock
├── src/tennis/
│   ├── ingestion/      # dlt sources & pipelines
│   ├── models/         # Elo, features, predictions (Polars)
│   ├── orchestration/  # Dagster defs, schedules, sensors
│   └── common/
├── dbt_project/        # models/{staging,intermediate,marts}, tests, seeds
├── apps/dashboard/     # Streamlit
├── data/{inbox,archive,warehouse}/   # gitignored
├── tests/
└── .github/workflows/
