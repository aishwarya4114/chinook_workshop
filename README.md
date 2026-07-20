# Chinook Data Warehouse Pipeline

A dimensional data warehouse built on Snowflake using the Chinook dataset (digital media store data — artists, albums, tracks, customers, and invoices), following Kimball methodology. The pipeline extracts data from SQL Server, stages it in Azure Blob Storage via Azure Data Factory, and transforms it into a star schema using dbt.

## Architecture

```
SQL Server (source)
      │
      ▼
Azure Data Factory  ──►  Azure Blob Storage (staging)
      │
      ▼
   Snowflake (raw layer)
      │
      ▼
   dbt (staging → intermediate → mart)
      │
      ▼
   Star Schema (fact + dimension tables)
```

## Tech Stack

- **Source / Orchestration:** SQL Server, Azure Data Factory
- **Storage:** Azure Blob Storage, Azure Key Vault (credential management), Managed Identity (storage access)
- **Warehouse:** Snowflake
- **Transformation:** dbt (staging, intermediate, and mart layers)
- **Modeling:** Kimball dimensional modeling (star schema)

## What This Project Does

1. **Extraction** — Azure Data Factory pulls data from 6 SQL Server tables (artists, albums, tracks, customers, invoices, invoice line items) and lands it in Azure Blob Storage.
2. **Credential Security** — Linked services are secured using Azure Key Vault, with Managed Identity handling storage access without hardcoded credentials.
3. **Loading** — Raw data is loaded into Snowflake as the base layer for transformation.
4. **Transformation (dbt)**
   - **Staging models** clean and standardize raw Snowflake tables (type casting, renaming, basic filtering).
   - **Intermediate models** apply business logic and join staging models where needed.
   - **Mart models** produce the final star schema — fact and dimension tables ready for analytics.
5. **Data Quality** — dbt tests (`not_null`, `unique`, `relationships`) are applied on primary and foreign key columns across staging and mart models to catch issues before they reach production schemas.
6. **Change Detection** — dbt's incremental materializations handle updated records without full table rebuilds, replacing a manual MD5 hash-comparison approach.

## Star Schema

- **Fact table:** `fct_invoices` (or similar) — grain at the invoice line level, with foreign keys to customer, track, and date dimensions.
- **Dimension tables:** `dim_customers`, `dim_tracks`, `dim_albums`, `dim_artists`, `dim_date` (5+ fact/dimension tables total).

## Data Quality & Testing

| Layer | Tests Applied |
|---|---|
| Staging | `not_null` on primary keys, `unique` on natural keys |
| Mart | `relationships` (foreign key integrity), `not_null`, `unique` |

Run tests locally:
```bash
dbt test
```

## Incremental Models

Fact tables use dbt's `incremental` materialization strategy to process only new or changed records on each run, rather than rebuilding the full table — improving run time and reducing warehouse compute cost as data volume grows.

```sql
{{
  config(
    materialized='incremental',
    unique_key='invoice_line_id'
  )
}}
```

## Project Structure

```
chinook-dbt/
├── models/
│   ├── staging/
│   │   ├── stg_customers.sql
│   │   ├── stg_invoices.sql
│   │   └── ...
│   ├── intermediate/
│   │   └── int_invoice_lines_joined.sql
│   └── marts/
│       ├── fct_invoices.sql
│       ├── dim_customers.sql
│       ├── dim_tracks.sql
│       ├── dim_albums.sql
│       └── dim_artists.sql
├── tests/
├── dbt_project.yml
└── README.md
```

## Setup

1. Configure Snowflake connection in `profiles.yml`.
2. Install dependencies:
   ```bash
   pip install dbt-snowflake
   ```
3. Run models:
   ```bash
   dbt run
   ```
4. Run tests:
   ```bash
   dbt test
   ```

