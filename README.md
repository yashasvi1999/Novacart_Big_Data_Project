> A production-grade, incremental data pipeline built on the **Medallion Architecture** using **Databricks**, **Delta Lake**, and **Azure SQL Database** — demonstrating real-world patterns like watermark-based ingestion, SCD Type 2, and automated orchestration.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Data Model](#data-model)
- [Pipeline Layers](#pipeline-layers)
  - [Bronze — Raw Layer](#-bronze--raw-layer)
  - [Silver — Clean Layer](#-silver--clean-layer)
  - [Gold — Business Layer](#-gold--business-layer)
- [Ingestion Strategy](#ingestion-strategy)
- [Orchestration](#orchestration)
- [Code Management](#code-management)
- [Alerts & Monitoring](#alerts--monitoring)
- [Key Concepts](#key-concepts)
- [Why This Matters](#why-this-matters)

---

## Overview

This project implements an **incremental data pipeline** for an e-commerce system that continuously generates and updates data across three core entities:

- `products`
- `orders`
- `payments`

The pipeline is designed to:

- Process **only new or updated records** — no full reloads
- Avoid **duplicate ingestion** across pipeline runs
- Maintain **state across runs** using control tables and watermark logic
- Produce **analytics-ready data** through a layered Medallion Architecture

---

## Architecture

```
Azure SQL Database
        │
        │  (Lakehouse Federation / Unity Catalog)
        ▼
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│    BRONZE     │────▶│    SILVER     │────▶│     GOLD      │
│   Raw Layer   │     │  Clean Layer  │     │ Business Layer│
│  Delta Tables │     │  Delta Tables │     │  Delta Tables │
└───────────────┘     └───────────────┘     └───────────────┘
        │                                                                                     
  Control Table
(Watermark State)                                                      
```

The pipeline is orchestrated end-to-end via **Databricks Workflows**, with all notebook code version-controlled through **GitHub + Databricks Repos**.

---

## Data Model

The pipeline operates on a relational e-commerce model:

| Entity     | Description                                      |
|------------|--------------------------------------------------|
| `products` | Product catalog with attributes and pricing      |
| `orders`   | Customer orders referencing products             |
| `payments` | Payment transactions linked to orders            |

These entities are **interconnected** — changes in one can impact downstream outputs. The Gold layer joins across all three to produce a unified, analytics-ready dataset.

---

## Pipeline Layers

### Bronze — Raw Layer

The Bronze layer is responsible for **raw ingestion** from the source system.

- Connects to **Azure SQL Database** via Databricks Lakehouse Federation (Unity Catalog)
- Uses **watermark logic** (timestamp + primary key) to identify new or updated records only
- Maintains a **control table** to persist the last processed state across runs
- Writes raw data into **Delta tables** with ingestion metadata (e.g., `_ingested_at`, `_source`)
- No transformations applied — data is stored as-is from the source

**Key pattern:** Watermark-based incremental load — only records where `updated_at > last_watermark` are ingested.

---

### Silver — Clean Layer

The Silver layer applies **cleaning, standardization, and quality checks**.

- Deduplicates records that may have been ingested multiple times
- Standardizes data types, formats, and naming conventions
- Applies **data quality checks** — invalid or non-conforming records are quarantined for review
- Performs **incremental upserts** into Silver Delta tables using `MERGE` (Delta Lake)
- Ensures only clean, trusted data moves downstream

**Key pattern:** Delta `MERGE` (upsert) — insert new records, update existing ones, skip unchanged rows.

---

### Gold — Business Layer

The Gold layer produces **analytics-ready, business-level datasets**.

- Joins `products`, `orders`, and `payments` into a unified output table
- Applies business logic and aggregations suited for reporting and analysis
- Implements **SCD Type 2** (Slowly Changing Dimensions) to preserve historical changes over time
- Data in this layer is optimized for consumption by BI tools and downstream applications

**Key pattern:** SCD Type 2 — historical rows are retained with `valid_from` / `valid_to` timestamps and an `is_current` flag.

---

## Ingestion Strategy

Data is sourced from **Azure SQL Database** using **Databricks Lakehouse Federation** via Unity Catalog.

- No traditional ETL connectors or data movement tools required
- Source tables are accessed directly through a Unity Catalog **connection** and **foreign catalog**
- Data is incrementally loaded into Bronze Delta tables using watermark-based logic

This approach reduces pipeline complexity and keeps data access governed and centralized within Unity Catalog.

---

## Orchestration

The pipeline is orchestrated using **Databricks Jobs / Workflows**.

- Notebook tasks (sourced from Databricks Repos) are chained in sequence: `Bronze → Silver → Gold`
- Each layer only runs after the previous one completes successfully
- Alerts are triggered at the end of the workflow for success/failure notifications
- The workflow is scheduled to run automatically, ensuring fresh data on a regular cadence

---

## Code Management

All pipeline code is version-controlled using **GitHub**, integrated with **Databricks Repos**.

```
repo/
├── bronze/
│   └── ingest_bronze.py
├── silver/
│   └── transform_silver.py
├── gold/
│   └── build_gold.py
├── utils/
│   └── watermark_utils.py
└── README.md
```

This setup enables:

- Full **version control** and change history
- **Collaboration** across team members via pull requests and code reviews
- Clean **environment separation** (dev / staging / prod branches)

---

## Alerts & Monitoring

Alerts are configured directly within Databricks Workflows to ensure pipeline reliability:

- **Failure alerts** — triggered immediately if any task in the workflow fails
- **Success alerts** — confirm end-to-end pipeline completion
- Alerts are sent via email (or connected notification channels)
- Observable at the task level — each Bronze / Silver / Gold step is individually monitored

---

## Key Concepts

| Concept | Description |
|---|---|
| **Incremental Loading** | Only new or changed records are processed per run |
| **Watermark Logic** | Tracks the last processed timestamp + primary key to determine new records |
| **Delta Lake MERGE** | Upserts records — inserts new rows, updates changed rows efficiently |
| **Control Table** | Stores pipeline state (e.g., last watermark) to enable restartability |
| **Idempotent Design** | Re-running the pipeline produces the same result without side effects |
| **SCD Type 2** | Preserves full history of record changes with effective date tracking |
| **Lakehouse Federation** | Direct, governed access to source databases via Unity Catalog — no data copies needed |

---

## Why This Matters

Most beginner data projects reload everything from scratch on every run. In production systems, that approach doesn't scale.

This project demonstrates the patterns that real data engineering teams use:

- Only **changes** are processed — efficient and cost-effective
- Pipelines are **version-controlled** — reproducible and auditable
- Workflows are **automated and chained** — no manual triggers
- **Monitoring and alerts** are built in — failures surface immediately
- Data **history is preserved** — SCD Type 2 tracks changes over time
- The pipeline is **idempotent** — safe to re-run without side effects

This is the kind of pipeline you'd find in a real data lakehouse environment — not just a demo.

---

*Built with Databricks · Delta Lake · Azure SQL · Unity Catalog · GitHub*
