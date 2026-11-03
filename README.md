# AWS Retail Data Engineering Pipeline

An end-to-end retail data engineering project built on AWS. The pipeline ingests retail CSV datasets, processes them through a Bronze–Silver–Gold medallion architecture, registers datasets in the AWS Glue Data Catalog, and uses Amazon Athena for SQL validation and business analytics.

> **Data model:** The implemented Gold layer contains three dimension tables and one fact table. Brand and category attributes are denormalised into `dim_products`; separate `dim_brands` and `dim_category` tables are not part of the implemented Gold ETL.

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
