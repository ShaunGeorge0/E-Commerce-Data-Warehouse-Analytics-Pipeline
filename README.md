# E-Commerce Data Platform

An end-to-end e-commerce data pipeline built on Databricks using the **Medallion Architecture** (Bronze → Silver → Gold). Raw CSV source data is ingested, cleaned, enriched, and consolidated into analytics-ready Delta tables under the `ecommerce` Unity Catalog.

## Architecture Overview

```
source_data (CSV volumes)
          │
          ▼
   ┌─────────────┐
   │  1_setup    │  Catalog & schema creation
   └─────────────┘
          │
          ▼
   ┌─────────────────────────────┐
   │  2_medallion_processing_dim  │
   │                             │
   │  Bronze: raw ingestion       │
   │  Silver: cleaning & standard │
   └─────────────────────────────┘
          │
          ▼
   ┌─────────────────────────────┐
   │  3_medallion_processing_fact │
   │  (planned — Gold layer)      │
   └─────────────────────────────┘
```

## Project Structure

```
data-proj/
├── 1_setup/
│   └── setup_catalog             # Creates the ecommerce catalog and bronze/silver/gold schemas
├── 2_medallion_processing_dim/
│   ├── 1_dim_bronze              # Bronze layer: raw CSV → Delta tables
│   └── 2_dim_silver             # Silver layer: cleaning, dedup, type casting, standardization
├── 3_medallion_processing_fact/  # (planned) Gold layer: dimension joins, fact tables, aggregations
└── README.md
```

## Data Layers

### Bronze Layer (Raw Ingestion)

**Notebook:** `2_medallion_processing_dim/1_dim_bronze`

Reads raw CSV files from Unity Catalog volumes (`/Volumes/ecommerce/source_data/raw/`) with explicitly defined schemas and writes them to Delta tables in `ecommerce.bronze`. Each table includes `_source_file` and `_ingested_at` metadata columns for traceability.

| Table | Source | Key Columns |
| --- | --- | --- |
| `brz_brands` | `brands/*.csv` | `brand_code`, `brand_name`, `category_code` |
| `brz_category` | `category/*.csv` | `category_code`, `category_name` |
| `brz_products` | `products/*.csv` | `product_id`, `sku`, `category_code`, `brand_code`, `color`, `size`, `material`, `weight_grams`, `length_cm`, `rating_count` |
| `brz_customers` | `customers/*.csv` | `customer_id`, `phone`, `country_code`, `country`, `state` |
| `brz_calendar` | `calendar/*.csv` | `date`, `year`, `day_name`, `quarter`, `week_of_year` |

### Silver Layer (Cleaning & Standardization)

**Notebook:** `2_medallion_processing_dim/2_dim_silver`

Reads from bronze, applies data-quality fixes, and writes to `ecommerce.silver`.

| Table | Bronze Source | Cleaning Operations |
| --- | --- | --- |
| `slv_brands` | `brz_brands` | Trim whitespace from `brand_name`; remove non-alphanumeric chars from `brand_code`; fix anomalous `category_code` values (GROCERY→GRCY, BOOKS→BKS, TOYS→TOY) |
| `slv_category` | `brz_category` | Remove duplicate `category_code` rows; uppercase `category_code` |
| `slv_products` | `brz_products` | Strip "g" suffix and cast `weight_grams` to integer; replace comma with dot and cast `length_cm` to float; uppercase `category_code` and `brand_code`; fix material spelling (Coton→Cotton, Alumium→Aluminum, Ruber→Rubber); convert negative `rating_count` to positive; null `rating_count` → 0 |
| `slv_customers` | `brz_customers` | Drop rows with null `customer_id`; fill null `phone` with "Not Available" |
| `slv_calendar` | `brz_calendar` | Convert `date` string to date type; remove duplicate rows by date; capitalize `day_name`; convert negative `week_of_year` to positive; format `quarter` as `Q{n}-{year}` and `week` as `Week{n}-{year}`; rename `week_of_year` to `week` |

### Gold Layer (Planned)

**Folder:** `3_medallion_processing_fact/` (not yet created)

The gold layer will join dimension tables, build fact tables, and produce analytics-ready aggregates for reporting and dashboards.

## Setup & Execution Order

1. Run `1_setup/setup_catalog` to create the `ecommerce` catalog and `bronze`, `silver`, `gold` schemas.
2. Run `2_medallion_processing_dim/1_dim_bronze` to ingest raw CSV data into bronze Delta tables.
3. Run `2_medallion_processing_dim/2_dim_silver` to clean and standardize data into silver Delta tables.
4. (Future) Run `3_medallion_processing_fact/` notebooks to build gold-layer fact tables and aggregations.

## Technologies

- Databricks (Serverless compute, PySpark, Databricks SQL)
- Delta Lake for storage format
- Unity Catalog for governance (`ecommerce` catalog)
- PySpark for data processing
- CSV source files stored in Unity Catalog Volumes

## Source Data

Raw CSV files are stored in `/Volumes/ecommerce/source_data/raw/` with separate subdirectories per entity (`brands/`, `category/`, `products/`, `customers/`, `calendar/`).