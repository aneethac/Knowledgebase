# BIW Enterprise Cloud Data Warehouse Architecture & Engineering Handbook

**Author:** Aneetha Chandhirasekar (Principal Data Engineer & Lead Data Architect)  
**Platform:** Snowflake, dbt Core/Cloud, Databricks, Oracle BIW Migration, Azure ADLS  
**Domain:** Supermarket Retail Analytics, Supply Chain & Enterprise BI  

---

## Table of Contents
1. [Executive Summary & Architectural Vision](#1-executive-summary--architectural-vision)
2. [6-Tier Decoupled Schema Topology](#2-6-tier-decoupled-schema-topology)
3. [Table Classifications, Naming Standards & Audit Rules](#3-table-classifications-naming-standards--audit-rules)
4. [Data Ingestion & Incremental Load Patterns](#4-data-ingestion--incremental-load-patterns)
5. [Kimball Dimensional Modeling & Star Schema Design](#5-kimball-dimensional-modeling--star-schema-design)
6. [Slowly Changing Dimensions (SCD Type 1 & Type 2)](#6-slowly-changing-dimensions-scd-type-1--type-2)
7. [Production dbt Project Architecture (`biw_cloud_dbt`)](#7-production-dbt-project-architecture-biw_cloud_dbt)
8. [End-to-End Pipeline Data Flow Walkthrough](#8-end-to-end-pipeline-data-flow-walkthrough)
9. [Zero-Downtime Blue-Green Cutover & Regression Parity](#9-zero-downtime-blue-green-cutover--regression-parity)
10. [System Tradeoffs, Optimization & Failure Modes Runbook](#10-system-tradeoffs-optimization--failure-modes-runbook)
11. [Data Lakehouse vs Data Warehouse Integration Layer](#11-data-lakehouse-vs-data-warehouse-integration-layer)

---

## 1. Executive Summary & Architectural Vision

The **BIW Enterprise Cloud Data Warehouse** is designed to support mission-critical business intelligence, financial reporting, supply chain optimization, and executive decision-making across 800+ supermarkets and retail convenience networks. Migrated from legacy Oracle BIW architecture to Snowflake and integrated with Databricks Delta Lake, this modern architecture provides:

- **Decoupled 6-Tier Layering**: Physical separation of ingestion staging, persisted historical CDC, transformation workspace, Kimball core dimensional storage, and consumer presentation layers.
- **Zero-Downtime Migration**: Blue-Green logical view repointing strategy ensuring zero disruption to downstream PowerBI dashboards, operational apps, and REST APIs during migration cutovers.
- **High-Throughput Incremental ELT**: dbt-driven incremental merge processing handling over 500 million daily records with automated data quality contracts.
- **Strict Data Governance & Parity**: Standardized surrogate key generation, enterprise audit columns, SCD Type 2 history tracking, and automated checksum regression validation across tens of millions of historical rows.

---

## 2. 6-Tier Decoupled Schema Topology

```
+-----------------------------------------------------------------------------------+
| 1. EDL (Enterprise Data Lake) - Azure ADLS Storage                                 |
| Raw JSON, CSV, Parquet files landed from upstream source systems (RMS, MRL, CML)  |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| 2. STG (Staging Layer) - Snowflake Transient Schema                                |
| Raw file copy landing, transient tables, schema-on-write validation               |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| 3. PSG (Persisted Staging Layer) - Snowflake Permanent Schema                      |
| Append-only immutable CDC history, source timestamping, deduplication storage     |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| 4. WRK / CR (Work & Change Tracking Layer) - Snowflake Transient Schema            |
| Incremental delta calculation, windowing functions, change detection hashing      |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| 5. BIW_DWH / BIW_DM (Core Warehouse & Data Marts) - Permanent Schema               |
| Kimball Star Schemas: Atomic Facts, SCD Type 1 & 2 Dimensions, Aggregates         |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| 6. OUT (Consumer Presentation Layer) - Snowflake Logical Secure Views             |
| Decoupled logical interface for PowerBI, Tableau, APIs, Databricks Delta Sharing   |
+-----------------------------------------------------------------------------------+
```

### Layer Specification Matrix

| Layer Name | Schema Type | Retention / Time Travel | Primary Purpose | Materialization Type |
|:---|:---|:---|:---|:---|
| **EDL** | ADLS Object Storage | 90 Days Blob Lifecycle | Landing raw files from operational systems | Object Files (Parquet/JSON) |
| **STG** | `TRANSIENT` | 0 Days | Fast initial `COPY INTO` ingest; temporary staging | `transient table` |
| **PSG** | `PERMANENT` | 30 Days | Immutable CDC landing & raw source history | `table` (Append-Only) |
| **WRK** | `TRANSIENT` | 1 Day | Pre-join windowing, delta calculation, deduplication | `transient table` |
| **BIW_DWH** | `PERMANENT` | 90 Days + Fail-Safe | Kimball Core (Dimensions, Facts, Aggregates) | `table` (Clustered) |
| **OUT** | View Layer | N/A | Business presentation, Security & Abstraction | `secure view` / `materialized view` |

---

## 3. Table Classifications, Naming Standards & Audit Rules

### Table Suffix Standards

- `*_STG`: Staging tables in `STG` layer.
- `*_TXN`: Transaction/event history tables in `PSG` layer.
- `*_REF`: Reference lookup tables.
- `*_WRK`: Work/processing tables in `WRK` layer.
- `*_DIM`: Dimension tables in `BIW_DWH` layer.
- `*_FCT`: Fact tables in `BIW_DWH` layer.
- `*_ATM`: Atomic level fact tables.
- `*_AGG`: Pre-calculated summary/aggregate tables.
- `*_REJ`: Rejected rows failing data quality validation.
- `*_MAP`: Mapping / cross-reference tables.

### Mandatory Column Class Words

- `*_KEY`: System-generated surrogate keys (e.g., `ITEM_KEY`, `LOCATION_KEY`).
- `*_IDNT`: Natural source business identifiers (e.g., `ITEM_IDNT`, `STORE_IDNT`).
- `*_CDE`: Categorical codes or status codes (e.g., `PROMO_TYPE_CDE`).
- `*_DESC`: Textual descriptions (e.g., `ITEM_DESC`).
- `*_FLAG`: Boolean or indicator flags (`Y`/`N` or `TRUE`/`FALSE`).
- `*_AMT`: Monitary financial amounts (e.g., `SALES_AMT`, `DISCOUNT_AMT`).
- `*_QTY`: Quantitative physical counts or weights (e.g., `SALES_QTY`).
- `*_DT`: Calendar date values (`YYYY-MM-DD`).
- `*_TS`: Timestamp values with timezone.

### Enterprise Audit Columns

Every table in `PSG` and `BIW_DWH` must include the following audit attributes:

```sql
CDW_RECD_LOAD_DT     TIMESTAMP_NTZ NOT NULL DEFAULT CURRENT_TIMESTAMP(),
CDW_RECD_LAST_UPD_DT TIMESTAMP_NTZ NOT NULL DEFAULT CURRENT_TIMESTAMP(),
CDW_RECD_CLOSE_DT    TIMESTAMP_NTZ DEFAULT TO_TIMESTAMP_NTZ('9999-12-31 23:59:59'),
CDW_RECD_CURR_FLG    VARCHAR(1)    NOT NULL DEFAULT 'Y',
CDW_PROCESS_KEY      NUMBER(38,0)  NOT NULL,
CDW_HASH_KEY         VARCHAR(64)   NOT NULL  -- SHA256 of business attributes
```

---

## 4. Data Ingestion & Incremental Load Patterns

### Bulk Ingestion (`COPY INTO`)

```sql
COPY INTO STG.STG_ITEM_LOCATION_SALES
FROM @EDL.AZURE_STAGE/retail/sales/raw/
FILE_FORMAT = (TYPE = 'PARQUET' COMPRESSION = 'SNAPPY')
PATTERN = '.*sales_.*\.parquet'
ON_ERROR = 'SKIP_FILE';
```

### CDC Incremental Merge Pattern (`PSG` -> `BIW_DWH`)

```sql
MERGE INTO BIW_DWH.FCT_ITEM_LOCATION_DAILY_SALES AS target
USING (
    SELECT 
        s.ITEM_KEY,
        s.LOCATION_KEY,
        s.SALES_DT,
        s.SALES_QTY,
        s.SALES_AMT,
        s.CDW_HASH_KEY,
        :CURRENT_PROCESS_KEY AS CDW_PROCESS_KEY
    FROM WRK.WRK_SALES_DELTA s
) AS src
ON target.ITEM_KEY = src.ITEM_KEY 
AND target.LOCATION_KEY = src.LOCATION_KEY 
AND target.SALES_DT = src.SALES_DT
WHEN MATCHED AND target.CDW_HASH_KEY <> src.CDW_HASH_KEY THEN
    UPDATE SET 
        target.SALES_QTY = src.SALES_QTY,
        target.SALES_AMT = src.SALES_AMT,
        target.CDW_HASH_KEY = src.CDW_HASH_KEY,
        target.CDW_RECD_LAST_UPD_DT = CURRENT_TIMESTAMP(),
        target.CDW_PROCESS_KEY = src.CDW_PROCESS_KEY
WHEN NOT MATCHED THEN
    INSERT (
        ITEM_KEY, LOCATION_KEY, SALES_DT, SALES_QTY, SALES_AMT,
        CDW_RECD_LOAD_DT, CDW_RECD_LAST_UPD_DT, CDW_RECD_CLOSE_DT, CDW_RECD_CURR_FLG, CDW_PROCESS_KEY, CDW_HASH_KEY
    )
    VALUES (
        src.ITEM_KEY, src.LOCATION_KEY, src.SALES_DT, src.SALES_QTY, src.SALES_AMT,
        CURRENT_TIMESTAMP(), CURRENT_TIMESTAMP(), TO_TIMESTAMP_NTZ('9999-12-31 23:59:59'), 'Y', src.CDW_PROCESS_KEY, src.CDW_HASH_KEY
    );
```

---

## 5. Kimball Dimensional Modeling & Star Schema Design

```
                     +---------------------------+
                     |    DIM_CALENDAR_DT        |
                     +---------------------------+
                     | CALENDAR_DT (PK)          |
                     | FIN_YEAR_NUMBER           |
                     | FIN_WEEK_NUMBER           |
                     +---------------------------+
                                   |
                                   | 1:N
                                   v
+-----------------------+    +------------------------------------+    +-----------------------+
|    DIM_ITEM_SCD2      |    | FCT_ITEM_LOCATION_DAILY_SALES_ATM  |    |   DIM_LOCATION_SCD2   |
+-----------------------+    +------------------------------------+    +-----------------------+
| ITEM_KEY (PK)         |    | SALES_EVENT_KEY (PK)               |    | LOCATION_KEY (PK)     |
| ITEM_IDNT (NK)        |--->| ITEM_KEY (FK)                      |<---| LOCATION_IDNT (NK)    |
| ITEM_DESC             | 1:N| LOCATION_KEY (FK)                  |1:N | STORE_NAME            |
| CATEGORY_CDE          |    | SALES_DT (FK)                      |    | ZONE_CDE              |
| EFF_START_DT          |    | SALES_QTY                          |    | EFF_START_DT          |
| EFF_END_DT            |    | SALES_AMT                          |    | EFF_END_DT            |
| CDW_RECD_CURR_FLG     |    | DISCOUNT_AMT                       |    | CDW_RECD_CURR_FLG     |
+-----------------------+    +------------------------------------+    +-----------------------+
```

---

## 6. Slowly Changing Dimensions (SCD Type 1 & Type 2)

### SCD Type 2 Merge SQL Logic

```sql
-- Step 1: Expire changed records
UPDATE BIW_DWH.DIM_ITEM_SCD2 target
SET 
    CDW_RECD_CLOSE_DT = src.EFFECTIVE_TS,
    CDW_RECD_CURR_FLG = 'N',
    CDW_RECD_LAST_UPD_DT = CURRENT_TIMESTAMP()
FROM WRK.WRK_ITEM_CHANGES src
WHERE target.ITEM_IDNT = src.ITEM_IDNT 
  AND target.CDW_RECD_CURR_FLG = 'Y'
  AND target.CDW_HASH_KEY <> src.CDW_HASH_KEY;

-- Step 2: Insert new current records
INSERT INTO BIW_DWH.DIM_ITEM_SCD2 (
    ITEM_KEY, ITEM_IDNT, ITEM_DESC, CATEGORY_CDE, BRAND_NAME,
    CDW_RECD_LOAD_DT, CDW_RECD_LAST_UPD_DT, CDW_RECD_CLOSE_DT, CDW_RECD_CURR_FLG, CDW_PROCESS_KEY, CDW_HASH_KEY
)
SELECT 
    HASH(src.ITEM_IDNT, src.EFFECTIVE_TS) AS ITEM_KEY,
    src.ITEM_IDNT,
    src.ITEM_DESC,
    src.CATEGORY_CDE,
    src.BRAND_NAME,
    src.EFFECTIVE_TS AS CDW_RECD_LOAD_DT,
    CURRENT_TIMESTAMP() AS CDW_RECD_LAST_UPD_DT,
    TO_TIMESTAMP_NTZ('9999-12-31 23:59:59') AS CDW_RECD_CLOSE_DT,
    'Y' AS CDW_RECD_CURR_FLG,
    src.CDW_PROCESS_KEY,
    src.CDW_HASH_KEY
FROM WRK.WRK_ITEM_CHANGES src;
```

---

## 7. Production dbt Project Architecture (`biw_cloud_dbt`)

```
biw_cloud_dbt/
├── dbt_project.yml
├── packages.yml
├── macros/
│   ├── generate_surrogate_key.sql
│   ├── scd2_handling.sql
│   └── checksum_validation.sql
├── models/
│   ├── staging/ (STG layer)
│   │   ├── stg_rms_items.sql
│   │   └── stg_sales_transactions.sql
│   ├── intermediate/ (WRK layer)
│   │   ├── int_item_hash_dedup.sql
│   │   └── int_sales_windowed.sql
│   ├── marts/ (BIW_DWH layer)
│   │   ├── dim_item_scd2.sql
│   │   ├── dim_location_scd2.sql
│   │   └── fct_item_location_daily_sales.sql
│   └── presentation/ (OUT layer)
│       ├── view_daily_sales_performance.sql
│       └── view_inventory_replenishment.sql
└── tests/
    ├── generic/
    │   └── test_scd2_overlap.sql
    └── singular/
        └── assert_sales_amt_non_negative.sql
```

### dbt CLI Execution Sequence
```bash
# Run staging & intermediate builds
dbt run --select staging intermediate

# Execute data quality tests on core warehouse models
dbt test --select marts

# Deploy presentation views
dbt run --select presentation
```

---

## 8. End-to-End Pipeline Data Flow Walkthrough

1. **Extraction & Landing**: Ingestion pipelines pull RMS transaction streams into Azure Data Lake Storage (`EDL`) as compressed Parquet files.
2. **Transient Copy**: Snowflake executes `COPY INTO` loading raw Parquet files into `STG.STG_SALES_TRANSACTIONS`.
3. **Persisted History**: Append-only CDC merge populates `PSG.PSG_SALES_TRANSACTIONS` with SHA256 business key hashing.
4. **Delta Windowing**: dbt transforms records into `WRK.WRK_SALES_WINDOWED`, executing `QUALIFY ROW_NUMBER() OVER (PARTITION BY TRANSACTION_ID ORDER BY SOURCE_TS DESC) = 1` for deduplication.
5. **Kimball Loading**: Surrogate key resolution maps dimension natural keys (`ITEM_IDNT`, `LOCATION_IDNT`) to surrogate dimension keys (`ITEM_KEY`, `LOCATION_KEY`), populating `BIW_DWH.FCT_ITEM_LOCATION_DAILY_SALES_ATM`.
6. **Consumer Presentation**: Downstream BI reports query secure views in `OUT` schema, completely isolated from underlying staging and physical storage schema migrations.

---

## 9. Zero-Downtime Blue-Green Cutover & Regression Parity

To migrate from legacy Oracle BIW to Snowflake without impacting 2,000+ business users:

1. **Parallel Run Validation**: Both Oracle BIW and Snowflake ran concurrently for 3 weeks (June 8–26, 2026).
2. **Checksum & Row-Count Parity**: Automated scripts calculated SHA256 checksums across fact metrics (`SALES_AMT`, `SALES_QTY`) grouped by business key. Parity target achieved: **0% variance across 50M+ historical records**.
3. **Logical Blue-Green View Repointing**:
   ```sql
   -- Cutover Command: Update presentation views to point to Snowflake core schema
   CREATE OR REPLACE SECURE VIEW OUT.PSG_ITEM_LOCATION_SALES AS
   SELECT 
       ITEM_IDNT,
       STORE_IDNT,
       SALES_DT,
       SALES_QTY,
       SALES_AMT
   FROM BIW_DWH.FCT_ITEM_LOCATION_DAILY_SALES_ATM;
   ```
4. **Outcome**: Zero downtime, zero application code changes required for BI consumption platforms.

---

## 10. System Tradeoffs, Optimization & Failure Modes Runbook

| Failure / Bottleneck | Root Cause | Architectural Mitigation / Resolution |
|:---|:---|:---|
| **Late-Arriving Facts** | Transaction arrived before dimension row created in `BIW_DWH` | **Inferred Member Pattern**: Insert placeholder dimension record with `ITEM_KEY = HASH('UNKNOWN')`, updated via retro-merge. |
| **SCD Type 2 Churn** | Frequent volatile attribute updates creating massive dimension explosion | **Outrigger Dimension Pattern**: Separate rapidly changing attributes into a Type 1 outrigger dimension. |
| **Out-of-Order CDC** | Ingestion pipeline re-ordering updates and deletes | **Windowing & Sequence Tracking**: Use `QUALIFY ROW_NUMBER() OVER (PARTITION BY ID ORDER BY CDW_SOURCE_TS DESC)` in `WRK` layer. |
| **Unclustered Fact Query Degredation** | Fact table growth causing micro-partition scanning slowdown | **Liquid Clustering**: Define `CLUSTER BY (SALES_DT, LOCATION_KEY)` on fact tables. |

---

## 11. Data Lakehouse vs Data Warehouse Integration Layer

```
+-------------------------------------------------------+
|             Azure ADLS Raw Landing Layer              |
+-------------------------------------------------------+
          |                                   |
          v                                   v
+------------------------------------+  +-------------------------------------+
|        Databricks Delta Lake       |  |          Snowflake DWH              |
|        (Lakehouse Platform)        |  |     (Cloud Data Warehouse)          |
+------------------------------------+  +-------------------------------------+
| - Delta Lake ACID Storage          |  | - Kimball Dimensional Star Schema   |
| - PySpark Feature Engineering      |  | - dbt ELT Data Pipeline Models      |
| - Point-in-Time (PIT) ML Features  |  | - Structured Financial Reporting    |
| - Databricks Unity Catalog         |  | - PowerBI Executive Dashboards      |
+------------------------------------+  +-------------------------------------+
          \                                   /
           \                                 /
            v                               v
+-------------------------------------------------------+
|    Delta Sharing & Snowflake Spark Interop Layer       |
|    Unified Single Source of Truth for ML & BI          |
+-------------------------------------------------------+
```
