# ShopVista Data Modernization Platform

A Bronze → Silver → Gold lakehouse pipeline on Azure Databricks that consolidates a fictional e-commerce company's scattered order, shipment, return, and customer data into a single analytics-ready warehouse feeding a Power BI dashboard.

## Problem Statement

ShopVista's order, shipment, return, and dimension data (customers, products, categories, brands, dates) lived in disconnected files and systems. That meant manual reconciliation before every report, delayed visibility into sales and returns, and no single source of truth for the business. This project designs and builds the platform that fixes that: automated ingestion, enforced data quality, and a star-schema warehouse ready for BI consumption.

## Architecture

![Pipeline Architecture](../architecture/pipeline_architecture.png)

Raw CSVs land in Azure Data Lake Storage Gen2, get picked up by Databricks Auto Loader, and move through three governed layers under Unity Catalog before reaching Power BI.

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Storage | Azure Data Lake Storage Gen2 (ADLS) | Centralized landing zone for raw files |
| Compute / ETL | Azure Databricks (PySpark, Structured Streaming) | Ingestion, transformation, aggregation |
| Governance | Unity Catalog | Catalog/schema access control, external volumes |
| Table format | Delta Lake | ACID transactions, merge/upsert, change data feed |
| Ingestion | Auto Loader (`cloudFiles`) | Incremental, schema-evolving file ingestion |
| BI | Power BI | Sales, revenue, and returns dashboards |

## Data Flow

1. **Landing** — Raw CSVs (order items, shipments, returns, plus dimension files for customers, products, categories, brands, dates) arrive in an ADLS container, organized by entity.
2. **Bronze** — Auto Loader streams each file into an append-only Delta table per entity, tagging every row with its source file path and ingestion timestamp. Nothing is cleaned here — Bronze is a faithful copy of what arrived.
3. **Silver** — Each Bronze table is deduplicated, type-cast, and standardized: string-encoded numerics get their units stripped (`"305g"` → `305`), inconsistent decimal separators are fixed, spelling anomalies in categorical fields are corrected against a lookup table, negative values are normalized, and nulls are handled explicitly (dropped for identifiers, filled for optional fields).
4. **Gold** — Silver tables are joined and enriched into star-schema fact and dimension tables: products are joined to brands and categories, customers are mapped to region via a country→state→region lookup, and the order-items fact table is enriched with calculated fields (gross amount, discount amount, net amount, coupon flag) using Delta's change data feed to process only inserts/updates. A rolling 30-day daily summary table sits on top for fast dashboard queries.
5. **Serving** — Power BI connects directly to the Gold layer for sales, revenue trend, and returns reporting.

## Key Engineering Decisions

- **Auto Loader + `trigger(availableNow=True)`** instead of a fully continuous stream — this is a daily-batch workload, not a low-latency one, so incremental batch processing gives the reliability of streaming (checkpointing, exactly-once semantics) without paying for always-on compute.
- **Change Data Feed on the Silver→Gold hop** for the fact table, so Gold only reprocesses actual inserts/updates from Silver instead of rescanning the whole table on every run.
- **MERGE (upsert) into Gold and Silver**, keyed on natural composite keys (`order_id + item_seq`), so re-running a load is idempotent instead of creating duplicates.
- **Medallion architecture with Unity Catalog schemas per layer** (`bronze` / `silver` / `gold`) rather than separate catalogs, so access policies and lineage stay simple to reason about.
- **Rolling 30-day summary table with merge-on-date** — recomputing the full history on every run doesn't scale, so the daily summary job only touches the last 30 days and merges by `date_id + currency`.

## Good to Know

**What I actually built**
- The full Bronze → Silver → Gold pipeline for all fact and dimension entities, using Auto Loader, Structured Streaming with `foreachBatch` upserts, and Delta MERGE.
- Data quality logic: null handling, deduplication, unit/format normalization, categorical standardization, negative-value correction.
- Star-schema Gold tables (products joined to brand/category, customers joined to region) and a rolling daily summary aggregate table.
- Unity Catalog setup: catalog, schemas, and an external volume backed by ADLS.

**What I understand but didn't implement here**
- *Row-level/column-level security in Unity Catalog:* "I'd apply row filters and column masks at the Unity Catalog table level so different business units only see the regions or fields they're cleared for, without duplicating tables."
- *CI/CD for notebook deployment:* "I'd use Databricks Asset Bundles with a GitHub Actions pipeline to promote notebooks from dev to prod workspaces instead of manually running them."
- *Data quality alerting:* "I'd wire up expectations (e.g., via Delta Live Tables or a custom check step) that fail the job or fire a Slack alert when null rates or row counts drift outside expected bounds, rather than relying on manual spot checks."
- *Full historical backfill automation:* "For a production system I'd parameterize the ingestion notebooks to backfill any date range on demand, rather than relying on `includeExistingFiles` picking up whatever's currently in the landing zone."

## Dashboard

![ShopVista Retail Intelligence Dashboard](../dashboard/ShopVista_dashboard.png)

Power BI dashboard surfacing total sales, repeat customer rate, sales by brand/category, customer distribution by region, channel split, and monthly revenue trend.


