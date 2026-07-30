# Netflix Data Engineering Pipeline

An end-to-end data engineering pipeline that ingests Netflix datasets from GitHub, processes them through the Medallion Architecture (Bronze → Silver → Gold), and delivers a governed, quality-checked Gold layer using Delta Live Tables.

## Overview

This project simulates a real-world content/media analytics pipeline: ingesting raw Netflix data files, transforming them through reusable, parameterized processing logic, and applying declarative data quality rules before the data reaches its final, analytics-ready form.

## Architecture

```
GitHub (source files)
        │
        │  Azure Data Factory: linked service → copy activity
        ▼
ADLS Gen2 — Bronze Layer (raw ingestion)
        │
        │  Databricks Autoloader: incremental file detection
        │  Databricks Notebook: parameterized cleaning (widgets + job task values)
        ▼
ADLS Gen2 — Silver Layer (cleaned, Delta format)
        │
        │  Delta Live Tables: streaming tables + declarative quality rules
        ▼
ADLS Gen2 — Gold Layer (validated, governed via Unity Catalog)
```
<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/e5ef5cdf-1c79-430b-87e2-dd386998e1de" />

## Tech Stack

| Layer | Technology |
|---|---|
| Source | GitHub (HTTP) |
| Orchestration | Azure Data Factory |
| Ingestion | Databricks Autoloader |
| Compute / Transformation | Azure Databricks, PySpark |
| Storage | Azure Data Lake Storage Gen2 (Delta Lake format) |
| Gold layer / data quality | Delta Live Tables (DLT) |
| Governance | Unity Catalog |
| Orchestration (jobs) | Databricks Workflows |

## Pipeline Stages

### 1. Bronze — Raw Ingestion
Azure Data Factory connects to a GitHub source via linked services and copies raw Netflix data files (titles, cast, directors, categories, countries) into ADLS Gen2. Databricks Autoloader is used to incrementally detect and load new files as they arrive, avoiding full reprocessing on every run.

### 2. Silver — Cleaning & Transformation
A single, reusable PySpark notebook processes each dataset. Rather than writing a separate script per dataset, the notebook accepts `sourcefolder` and `targetfolder` as parameters (via `dbutils.widgets`), reads the corresponding CSV files from Bronze, and writes cleaned Delta tables to the Silver layer. A companion notebook demonstrates passing an array of dataset configurations through `dbutils.jobs.taskValues`, allowing the same notebook to be triggered across multiple datasets from a Databricks Workflow without duplicating code.

### 3. Gold — Delta Live Tables & Data Quality
The Gold layer is built using Delta Live Tables, reading each Silver dataset as a streaming source. Declarative data quality expectations (`@dlt.expect_all_or_drop`) enforce record-level rules — for example, dropping records with a null `show_id` — before the data is exposed as a governed Gold table. A staging-and-transformation pattern is used for the titles dataset specifically: a staging DLT table reads the raw stream, an intermediate view applies a transformation, and a final DLT table applies quality rules before producing the validated output.

### 4. Governance & Orchestration
Gold-layer tables are organized under Unity Catalog for centralized governance and access control. Databricks Workflows schedule and orchestrate the overall job execution.

## Key Design Decisions

**Parameterized, reusable notebooks over per-dataset scripts.** Instead of duplicating transformation logic for each of the five datasets, a single Silver notebook is parameterized by folder name, and a companion notebook demonstrates driving that logic across multiple datasets via Databricks job parameters — reducing code duplication and maintenance overhead.

**Declarative data quality via Delta Live Tables.** Rather than writing imperative validation logic (explicit row counts, manual filtering), DLT's `expect_all_or_drop` expectations declare the quality rule once and let the DLT runtime handle enforcement and dropped-record tracking automatically.

## Repository Structure

```
netflix-data-engineering-pipeline/
├── adf-pipelines/
│   └── (ADF pipeline + linked service JSON files)
├── databricks-notebooks/
│   ├── bronze_to_silver_data_transfer.ipynb
│   ├── lookup_array_parameter_setup.ipynb
│   ├── weekday_variable_input_and_job_output_assignment.ipynb
│   ├── weekday_lookup_task_output_retrieval.ipynb
│   └── gold_layer_data_quality_checks.ipynb
├── sample_data/
│   └── (sample Netflix CSV files)
└── README.md
```

## Possible Extensions

- Extend row-level data quality expectations beyond null checks (e.g., referential integrity between titles and cast/directors)
- Add a Bronze-to-Silver row-count reconciliation step for auditability
- Parameterize the ADF pipeline to dynamically discover new dataset folders from the GitHub source, similar to the metadata-driven table discovery used in a related fintech migration project

## Author

Soujanya Mandula — [LinkedIn](https://www.linkedin.com/in/soujanyamandula/)
