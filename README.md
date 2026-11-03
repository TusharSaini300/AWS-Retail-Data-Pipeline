# AWS Retail Data Engineering Pipeline

An end-to-end retail data engineering project built on AWS. The pipeline ingests retail CSV datasets, processes them through a Bronze–Silver–Gold medallion architecture, registers datasets in the AWS Glue Data Catalog, and uses Amazon Athena for SQL validation and business analytics.

> **Data Model:** The implemented Gold layer contains three dimension tables and one fact table. Brand and category attributes are denormalised into `dim_products`; separate `dim_brands` and `dim_category` tables are not part of the implemented Gold ETL.

## Project Overview

Retail data is spread across multiple source files and requires cleaning, standardisation, integration, and modelling before it can support analysis. This project builds a cloud-based data lake and ETL workflow to prepare retail data for analytical queries.

### Objectives

- Ingest retail CSV datasets into Amazon S3.
- Convert raw CSV data into Parquet format.
- Clean, standardise, and validate records using AWS Glue and PySpark.
- Organise processed data using Bronze, Silver, and Gold layers.
- Build a dimensional model for retail transactions.
- Query and validate curated data using Amazon Athena.
- Create a denormalised business analytics dataset using Athena CTAS.

## Architecture

The implemented workflow:

```text
Retail CSV Sources
       |
       v
Amazon S3 - Raw
       |
       v
AWS Glue - Bronze ETL
       |
       v
Amazon S3 - Bronze (Parquet)
       |
       v
AWS Glue - Silver ETL
       |----------------------> Rejected Records
       v
Amazon S3 - Silver (Parquet)
       |
       v
AWS Glue - Gold ETL
       |
       v
Amazon S3 - Gold (Parquet)
       |
       v
AWS Glue Crawlers
       |
       v
AWS Glue Data Catalog
       |
       v
Amazon Athena
       |----------------------> Athena Query Results
       |
       v
CTAS: gld_business_analytics
       |
       v
Amazon S3 - gold/business_analytics/
       |
       v
Business Analytics Dataset
```

## Technology Stack

| Technology | Purpose |
|---|---|
| Amazon S3 | Data lake storage |
| AWS Glue ETL | PySpark-based data ingestion and transformations |
| AWS Glue Crawlers | Schema discovery and table registration |
| AWS Glue Data Catalog | Central metadata catalogue |
| Amazon Athena | SQL querying, validation, and CTAS |
| Python / PySpark | ETL transformation logic |
| Parquet | Columnar storage format |
| Power BI Desktop | Business intelligence and dashboard layer |

## Source Datasets

The pipeline uses six retail datasets:

| Dataset | Description |
|---|---|
| Brands | Brand reference data |
| Category | Product category reference data |
| Products | Product attributes |
| Customers | Customer and geographic attributes |
| Date | Calendar attributes |
| Order Items | Transaction line-item data, supplied as 92 daily CSV files |

The source data contains inconsistencies such as whitespace, inconsistent casing, duplicate records, malformed numeric values, missing values, and inconsistent date/time formats.

## Medallion Architecture

### 1. Bronze Layer — Raw Ingestion

**Glue Job:** `retail-bronze-etl`

The Bronze job reads the source CSV datasets from S3, adds ingestion metadata, and writes the datasets as Parquet files.

Key operations:

- Read CSV files with headers.
- Preserve source fields as strings during initial ingestion.
- Add `ingestion_timestamp`.
- Add `source_file`.
- Write Parquet datasets to the Bronze S3 prefixes.

The Bronze layer preserves the source structure while preparing the data for downstream processing.

### 2. Silver Layer — Cleaning and Standardisation

**Glue Job:** `retail-silver-etl`

The Silver job applies data-quality and standardisation rules to the Bronze datasets.

Key transformations:

- Trim and standardise text fields.
- Standardise brand and category codes.
- Correct inconsistent material values.
- Convert weight and dimension fields into numeric types.
- Handle missing customer phone values.
- Remove duplicate category, date, and transaction line records.
- Standardise date and timestamp formats.
- Clean quantity, unit price, discount, and tax fields.
- Standardise sales channel values.
- Route rejected records to a separate data-quality location.

**Rejected Records Location:**

`s3://tus-retail-clothing-store-datasets/data_quality/rejected_records/`

### 3. Gold Layer — Dimensional Modelling

**Glue Job:** `retail-gold-etl`

The Gold job creates the analytical fact and dimension tables.

| Table | Type | Description |
|---|---|---|
| `dim_products` | Dimension | Product attributes, including brand and category names |
| `dim_customers` | Dimension | Customer and geographic region attributes |
| `dim_date` | Dimension | Calendar attributes for time-based analysis |
| `fact_order_items` | Fact | Transaction line items and financial measures |

The fact table grain is one order-item record. The transaction and item sequence identify an individual line item.

**Modelling detail:** Brand and category attributes are included in `dim_products`. The current implementation does not create separate `dim_brands` or `dim_category` Gold tables.

## Gold Star Schema

The implemented Gold model follows a star schema with one central fact table and three dimensions.

```text
                 dim_date
                    |
                    |
dim_customers -- fact_order_items -- dim_products
```

### Fact Table: `fact_order_items`

Contains transaction identifiers, foreign keys, sales channel, quantity, unit price, gross amount, discount, tax, net amount, and INR-converted net amount.

### Dimension Tables

- `dim_date`: Date, year, day name, quarter, week, and weekend indicator.
- `dim_customers`: Customer attributes and derived geographic region.
- `dim_products`: Product attributes, brand details, and category details.

## Athena and Business Analytics

The Gold tables are registered in the Glue Data Catalog and queried through Amazon Athena.

The business analytics dataset is created using an Athena CTAS statement that:

- Reads `fact_order_items`.
- Joins `dim_date` using `date_id`.
- Joins `dim_products` using `product_id`.
- Selects date and product attributes alongside fact columns.
- Extracts `hour_of_day` from `transaction_ts`.
- Writes the resulting dataset in Parquet format.

**Analytics Table:** `retail_gold_db.gld_business_analytics`

**CTAS Output Location:**

`s3://tus-retail-clothing-store-datasets/gold/business_analytics/`

**Athena Query Results Location:**

`s3://tus-retail-clothing-store-datasets/athena-results/`

These are separate locations: CTAS writes the business analytics dataset, while Athena's query-results location stores query execution output.

The current denormalisation query joins the fact table to the date and product dimensions. It does not join `dim_customers`, so customer geography is not included in `gld_business_analytics`.
