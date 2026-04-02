# Data Pipeline Architecture

## High-Level Overview

```text
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              ORCHESTRATION (Airflow / Dagster)                       │
│                                                                                     │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐   │
│  │   SOURCES    │     │   STAGING    │     │   STORAGE    │     │  CONSUMPTION  │   │
│  │              │     │              │     │              │     │              │   │
│  │  APIs        │────▶│  S3 / GCS /  │────▶│  Warehouse   │────▶│  BI Tools    │   │
│  │  OLTP (RDS)  │     │  ADLS        │     │  or          │     │  Notebooks   │   │
│  │  3rd Party   │     │  (Raw Zone)  │     │  Lakehouse   │     │  BI as Code  │   │
│  │  Web Scraper │     │              │     │              │     │  ML Models   │   │
│  └──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘   │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                        CI/CD + CONTAINERIZATION                              │   │
│  │                     GitHub Actions / Docker / Terraform                       │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Detailed Flow

```text
                    EXTRACT                    LOAD                 TRANSFORM
                 ┌───────────┐            ┌───────────┐          ┌───────────┐
                 │           │            │           │          │           │
  ┌────────┐    │  Ingest   │   ┌─────┐  │  Land in  │  ┌────┐  │  Model &  │   ┌─────┐
  │ Sources │───▶│  Layer    │──▶│ S3  │──▶│  Target   │──▶│ dbt│──▶│  Serve    │──▶│ BI  │
  └────────┘    │ (Airbyte, │   │ GCS │  │ (Snowflake│  └────┘  │           │   └─────┘
                │  Fivetran, │   │ADLS │  │  Redshift,│          └───────────┘
                │  dlt,      │   └─────┘  │  BigQuery,│
                │  custom)   │            │  Lakehouse│
                └───────────┘            └───────────┘
```

## Warehouse vs Lakehouse

```text
  WAREHOUSE (Snowflake, Redshift, BigQuery)       LAKEHOUSE (S3 + Trino + Iceberg + Catalog)
  ┌─────────────────────────────────┐              ┌─────────────────────────────────────┐
  │  Managed compute + storage      │              │  Object Storage (S3)                │
  │  SQL-native                     │              │     + Table Format (Iceberg/Delta)   │
  │  Pay per query / warehouse      │              │     + Query Engine (Trino/Spark)     │
  │  Less operational overhead      │              │     + Catalog (Postgres/Hive/Glue)   │
  │  Vendor lock-in                 │              │  Open formats, more control          │
  │                                 │              │  More operational overhead            │
  └─────────────────────────────────┘              └─────────────────────────────────────┘
```
