# BIW Enterprise Cloud Data Warehouse — Architectural Handbook & Interview Guide

> **Author:** Aneetha Chandhirasekar — Principal Data Engineer | Lead Data & Solution Architect
> **Certification:** Databricks Certified Data Engineer Associate
> **Domain:** Retail Supply Chain Analytics | AI/ML Feature Stores | Enterprise Cloud Data Warehousing
> **Last Updated:** August 2026

---

## Table of Contents

1. [System Architecture & 6-Tier Schema Topology](#1-system-architecture--6-tier-schema-topology)
2. [Table Classifications & Naming Conventions](#2-table-classifications--naming-conventions)
3. [Data Ingestion & Incremental Load Patterns](#3-data-ingestion--incremental-load-patterns)
4. [Kimball Dimensional Modeling Standards](#4-kimball-dimensional-modeling-standards)
5. [Slowly Changing Dimensions (SCD) Implementation](#5-slowly-changing-dimensions-scd-implementation)
6. [dbt Project Structure for BIW ELT Models](#6-dbt-project-structure-for-biw-elt-models)
7. [Full End-to-End Pipeline Demo Walkthrough](#7-full-end-to-end-pipeline-demo-walkthrough)
8. [System Design Tradeoffs & Failure Modes Runbook](#8-system-design-tradeoffs--failure-modes-runbook)
9. [Lakehouse vs Data Warehouse — Key Concepts](#9-lakehouse-vs-data-warehouse--key-concepts)
10. [STAR Interview Playbook — Project Atlantis](#10-star-interview-playbook--project-atlantis)
11. [Self-Introduction Scripts — Principal Data Engineer Profile](#11-self-introduction-scripts--principal-data-engineer-profile)

---

## 1. System Architecture & 6-Tier Schema Topology

### Overview

The BIW Cloud Data Warehouse is built on a **decoupled 6-tier schema topology** on Snowflake. This architecture isolates each data processing concern so that upstream schema changes never cascade to downstream consumers.

```
Azure Data Lake (EDL)
        │
        ▼
   STG (Staging)          ← Transient landing from COPY INTO
        │
        ▼
   PSG (Persisted Staging) ← Raw CDC history retained
        │
        ▼
   WRK / CR (Work Layer)  ← Deduplication, window functions, surrogate keys
        │
        ▼
   BIW_DWH / DM (Core)   ← Kimball Star Schema (dim_*, fct_*)
        │
        ▼
   OUT / Views (Consumer) ← Logical views for BI tools and Feature Stores
```

### Layer Definitions

| Layer | Schema | Materialization | Storage Type | Purpose |
|:------|:-------|:----------------|:-------------|:--------|
| **EDL** | External Stage | Stage | Azure ADLS | Raw file landing (CSV, Parquet, Delta) |
| **STG** | `STG` | Transient Table | TRANSIENT | Fast COPY INTO from stage, truncated after load |
| **PSG** | `PSG` | Permanent Table | PERMANENT | Persisted raw CDC history with audit columns |
| **WRK** | `WRK` | Transient Table | TRANSIENT | Intermediate deduplication and key resolution |
| **CR** | `CR` | Permanent Table | PERMANENT | Common reference data, lookup tables |
| **DM / BIW_DWH** | `BIW_DWH` | Permanent Table | PERMANENT | Conformed Kimball Star Schema dimensions & facts |
| **OUT** | `OUT` | View | N/A | Consumer abstraction views for BI and ML |

### Time Travel & Fail-Safe Configuration

| Layer | Time Travel Retention | Fail-Safe |
|:------|:----------------------|:----------|
| STG | 0 days | Disabled |
| PSG | 14 days | Enabled |
| BIW_DWH | 90 days | Enabled |
| OUT Views | N/A | N/A |

---

## 2. Table Classifications & Naming Conventions

### Table Suffix Standards

| Suffix | Layer | Description |
|:-------|:------|:------------|
| `*_STG` | STG | Raw staging table |
| `*_TXN` | PSG | CDC transaction history table |
| `*_REF` | CR | Reference / lookup table |
| `*_WRK` | WRK | Intermediate work table |
| `*_DIM` | BIW_DWH | Conformed dimension table |
| `*_ATM` | BIW_DWH | Atomic fact table |
| `*_AGG` | BIW_DWH | Aggregated fact table |
| `*_REJ` | WRK | Reject / error table |
| `*_MAP` | CR | Mapping / crosswalk table |

### Mandatory Column Class Words

| Class Word | Data Type | Description | Example |
|:-----------|:----------|:------------|:--------|
| `_KEY` | NUMBER | Surrogate key (hash or sequence) | `PROD_ITEM_KEY` |
| `_IDNT` | VARCHAR | Natural / business identifier | `ITEM_IDNT` |
| `_CDE` | VARCHAR | Code value | `CATEGORY_CDE` |
| `_DESC` | VARCHAR | Description | `ITEM_DESC` |
| `_FLG` | CHAR(1) | Boolean flag (Y/N) | `CDW_RECD_CURR_FLG` |
| `_AMT` | NUMBER | Monetary amount | `UNIT_RETAIL_AMT` |
| `_QTY` | NUMBER | Quantity | `SALES_QTY` |
| `_DT` | DATE | Calendar date | `TRAN_DT` |
| `_TS` | TIMESTAMP_NTZ | Timestamp (UTC) | `CDW_RECD_LOAD_TS` |

### Standard Audit Columns (All BIW_DWH Tables)

```sql
CDW_RECD_LOAD_DT        TIMESTAMP_NTZ   -- Row insert timestamp
CDW_RECD_LAST_UPD_DT    TIMESTAMP_NTZ   -- Row last updated timestamp
CDW_RECD_CLOSE_DT       TIMESTAMP_NTZ   -- SCD2 row expiry timestamp
CDW_RECD_CURR_FLG       CHAR(1)         -- 'Y' = active, 'N' = expired
CDW_PROCESS_KEY         NUMBER          -- Pipeline execution batch key
CDW_SOURCE_SYSTEM_CDE   VARCHAR(10)     -- Source system identifier
```

---

## 3. Data Ingestion & Incremental Load Patterns

### Pattern 1 — Initial Historical Bulk Load

```sql
-- Step 1: Create file format
CREATE OR REPLACE FILE FORMAT BIW_PARQUET_FORMAT
    TYPE = 'PARQUET'
    SNAPPY_COMPRESSION = TRUE;

-- Step 2: Bulk COPY INTO staging
COPY INTO STG.RMS_PROD_ITEM_STG
FROM @BIW_AZURE_STAGE/rms/prod_item/
FILE_FORMAT = (FORMAT_NAME = 'BIW_PARQUET_FORMAT')
ON_ERROR = 'CONTINUE'
PURGE = FALSE;
```

### Pattern 2 — CDC Incremental Merge

```sql
-- Step 1: Land delta records into STG
COPY INTO STG.RMS_PROD_ITEM_STG
FROM @BIW_AZURE_STAGE/rms/prod_item/delta/
FILE_FORMAT = (FORMAT_NAME = 'BIW_PARQUET_FORMAT')
ON_ERROR = 'CONTINUE';

-- Step 2: Merge CDC into PSG (persistent raw history)
MERGE INTO PSG.RMS_PROD_ITEM_TXN AS TGT
USING STG.RMS_PROD_ITEM_STG AS SRC
ON TGT.ITEM_IDNT = SRC.ITEM_IDNT
   AND TGT.CDW_PROCESS_KEY = SRC.CDW_PROCESS_KEY
WHEN NOT MATCHED THEN INSERT (
    ITEM_IDNT, ITEM_DESC, CATEGORY_CDE, UNIT_RETAIL_AMT,
    CDW_RECD_LOAD_DT, CDW_PROCESS_KEY, CDW_SOURCE_SYSTEM_CDE
) VALUES (
    SRC.ITEM_IDNT, SRC.ITEM_DESC, SRC.CATEGORY_CDE, SRC.UNIT_RETAIL_AMT,
    CURRENT_TIMESTAMP(), SRC.CDW_PROCESS_KEY, 'RMS'
);
```

### Pattern 3 — dbt Incremental Watermark

```sql
-- models/marts/core/fct_sales_transaction_plu_atm.sql
{{
    config(
        materialized='incremental',
        unique_key='SALES_TRAN_KEY',
        incremental_strategy='merge',
        on_schema_change='append_new_columns',
        cluster_by=['TRAN_DT', 'ORG_LOC_KEY']
    )
}}

SELECT
    {{ dbt_utils.generate_surrogate_key(['TRAN_IDNT', 'ITEM_IDNT', 'ORG_LOC_IDNT']) }} AS SALES_TRAN_KEY,
    s.TRAN_IDNT,
    s.TRAN_TS,
    p.PROD_ITEM_KEY,
    l.ORG_LOC_KEY,
    s.SALES_QTY,
    s.RETAIL_AMT,
    CURRENT_TIMESTAMP() AS CDW_RECD_LOAD_DT
FROM PSG.POS_SALES_TXN_TXN s
LEFT JOIN {{ ref('dim_prod_item') }} p
    ON s.ITEM_IDNT = p.ITEM_IDNT
    AND s.TRAN_TS >= p.CDW_RECD_LOAD_DT
    AND s.TRAN_TS < COALESCE(p.CDW_RECD_CLOSE_DT, '9999-12-31'::TIMESTAMP_NTZ)
LEFT JOIN {{ ref('dim_org_loc') }} l
    ON s.ORG_LOC_IDNT = l.ORG_LOC_IDNT
    AND l.CDW_RECD_CURR_FLG = 'Y'

{% if is_incremental() %}
WHERE s.CDW_PROCESS_KEY > (SELECT MAX(CDW_PROCESS_KEY) FROM {{ this }})
{% endif %}
```

---

## 4. Kimball Dimensional Modeling Standards

### Star Schema Design Rules

1. **Fact Tables** — store measurable, numeric events at atomic grain
2. **Dimension Tables** — store descriptive context for fact analysis
3. **Conformed Dimensions** — shared across multiple subject area facts
4. **Surrogate Keys** — all dimension joins via system-generated hash keys
5. **Atomic Grain** — `Product-Item` × `Location` × `Day` (PLU-ATM level)

### Fact Table Standards

```sql
-- Atomic Fact Table Example
CREATE OR REPLACE TABLE BIW_DWH.FCT_SALES_TRANSACTION_PLU_ATM (
    SALES_TRAN_KEY          NUMBER NOT NULL,        -- Surrogate Key
    TRAN_DT                 DATE NOT NULL,          -- Degenerate Dimension
    PROD_ITEM_KEY           NUMBER NOT NULL,        -- FK → DIM_PROD_ITEM
    ORG_LOC_KEY             NUMBER NOT NULL,        -- FK → DIM_ORG_LOC
    SALES_QTY               NUMBER(15,3),           -- Additive Fact
    RETAIL_AMT              NUMBER(15,2),           -- Additive Fact
    DISCOUNT_AMT            NUMBER(15,2),           -- Additive Fact
    CDW_RECD_LOAD_DT        TIMESTAMP_NTZ,
    CDW_PROCESS_KEY         NUMBER
)
CLUSTER BY (TRAN_DT, ORG_LOC_KEY);
```

### Dimension Table Standards

```sql
-- Conformed Dimension Example (SCD Type 2)
CREATE OR REPLACE TABLE BIW_DWH.DIM_PROD_ITEM (
    PROD_ITEM_KEY           NUMBER NOT NULL,        -- Surrogate Key (PK)
    ITEM_IDNT               VARCHAR(20) NOT NULL,   -- Natural Key
    ITEM_DESC               VARCHAR(200),
    CATEGORY_CDE            VARCHAR(10),
    CATEGORY_DESC           VARCHAR(100),
    DEPT_CDE                VARCHAR(10),
    UNIT_RETAIL_AMT         NUMBER(10,2),
    IS_MERCH_RANGED_FLG     CHAR(1),
    IS_SUPPLY_ACTIVE_FLG    CHAR(1),
    CDW_RECD_LOAD_DT        TIMESTAMP_NTZ,
    CDW_RECD_CLOSE_DT       TIMESTAMP_NTZ,
    CDW_RECD_CURR_FLG       CHAR(1) DEFAULT 'Y',
    CDW_PROCESS_KEY         NUMBER
);
```

---

## 5. Slowly Changing Dimensions (SCD) Implementation

### SCD Type 1 — Overwrite (No History)

Used for: Corrections, typos, non-analytically significant attribute changes.

```sql
MERGE INTO BIW_DWH.DIM_PROD_ITEM AS TGT
USING WRK.PROD_ITEM_WRK AS SRC
ON TGT.ITEM_IDNT = SRC.ITEM_IDNT
   AND TGT.CDW_RECD_CURR_FLG = 'Y'
WHEN MATCHED AND (
    TGT.ITEM_DESC <> SRC.ITEM_DESC OR
    TGT.CATEGORY_CDE <> SRC.CATEGORY_CDE
) THEN UPDATE SET
    TGT.ITEM_DESC       = SRC.ITEM_DESC,
    TGT.CATEGORY_CDE    = SRC.CATEGORY_CDE,
    TGT.CDW_RECD_LAST_UPD_DT = CURRENT_TIMESTAMP();
```

### SCD Type 2 — History Tracking (Full Versioning)

Used for: Analytically significant attribute changes (price, category reclassification, range status).

```sql
-- Step 1: Expire active row
UPDATE BIW_DWH.DIM_PROD_ITEM
SET CDW_RECD_CURR_FLG  = 'N',
    CDW_RECD_CLOSE_DT  = CURRENT_TIMESTAMP(),
    CDW_RECD_LAST_UPD_DT = CURRENT_TIMESTAMP()
WHERE ITEM_IDNT IN (
    SELECT SRC.ITEM_IDNT
    FROM WRK.PROD_ITEM_WRK SRC
    JOIN BIW_DWH.DIM_PROD_ITEM TGT
        ON SRC.ITEM_IDNT = TGT.ITEM_IDNT
        AND TGT.CDW_RECD_CURR_FLG = 'Y'
    WHERE SRC.CATEGORY_CDE <> TGT.CATEGORY_CDE
       OR SRC.UNIT_RETAIL_AMT <> TGT.UNIT_RETAIL_AMT
)
AND CDW_RECD_CURR_FLG = 'Y';

-- Step 2: Insert new active row
INSERT INTO BIW_DWH.DIM_PROD_ITEM (
    PROD_ITEM_KEY, ITEM_IDNT, ITEM_DESC, CATEGORY_CDE,
    UNIT_RETAIL_AMT, CDW_RECD_LOAD_DT, CDW_RECD_CLOSE_DT,
    CDW_RECD_CURR_FLG, CDW_PROCESS_KEY
)
SELECT
    {{ dbt_utils.generate_surrogate_key(['ITEM_IDNT', 'CURRENT_TIMESTAMP()']) }},
    SRC.ITEM_IDNT, SRC.ITEM_DESC, SRC.CATEGORY_CDE,
    SRC.UNIT_RETAIL_AMT, CURRENT_TIMESTAMP(), '9999-12-31'::TIMESTAMP_NTZ,
    'Y', SRC.CDW_PROCESS_KEY
FROM WRK.PROD_ITEM_WRK SRC
WHERE SRC.ITEM_IDNT IN (
    SELECT ITEM_IDNT FROM BIW_DWH.DIM_PROD_ITEM
    WHERE CDW_RECD_CURR_FLG = 'N'
    AND CDW_RECD_CLOSE_DT >= CURRENT_TIMESTAMP() - INTERVAL '1 minute'
);
```

---

## 6. dbt Project Structure for BIW ELT Models

### Full Project Folder Tree

```
biw_cloud_dbt/
├── dbt_project.yml
├── packages.yml
├── profiles.yml
├── macros/
│   ├── generate_surrogate_key.sql
│   ├── scd2_merge_helper.sql
│   └── grant_consumer_privs.sql
├── models/
│   ├── staging/
│   │   └── rms/
│   │       ├── stg_rms__prod_item.sql
│   │       └── stg_rms__prod_item.yml
│   ├── intermediate/
│   │   ├── int_rms__prod_item_cdc.sql
│   │   └── int_sales__tran_item_wrk.sql
│   └── marts/
│       ├── core/
│       │   ├── dim_prod_item.sql
│       │   ├── dim_org_loc.sql
│       │   ├── fct_sales_transaction_plu_atm.sql
│       │   └── core_marts.yml
│       └── exposure_views/
│           └── v_dim_prod_item_curr.sql
├── snapshots/
│   └── snp_rms__prod_item.sql
└── tests/
    ├── assert_fact_sales_amount_positive.sql
    └── assert_scd2_no_overlapping_dates.sql
```

### dbt_project.yml — Schema Routing

```yaml
name: 'biw_cloud_dbt'
version: '1.0.0'
profile: 'biw_snowflake_profile'

models:
  biw_cloud_dbt:
    staging:
      +database: BIW_DEV
      +schema: STG
      +materialized: view
    intermediate:
      +database: BIW_DEV
      +schema: WRK
      +materialized: ephemeral
    marts:
      core:
        +database: BIW_DEV
        +schema: BIW_DWH
        +materialized: incremental
        +on_schema_change: "append_new_columns"
      exposure_views:
        +database: BIW_DEV
        +schema: BIW_DWH
        +materialized: view
```

### packages.yml

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.0.0", "<2.0.0"]
  - package: calogica/dbt_expectations
    version: [">=0.8.0", "<1.0.0"]
  - package: dbt-labs/snowflake_utils
    version: [">=0.1.0", "<1.0.0"]
```

### Key Model Examples

#### Staging Model — `stg_rms__prod_item.sql`

```sql
-- models/staging/rms/stg_rms__prod_item.sql
{{
    config(materialized='view', schema='STG')
}}

SELECT
    ITEM_IDNT::VARCHAR(20)          AS ITEM_IDNT,
    ITEM_DESC::VARCHAR(200)         AS ITEM_DESC,
    CATEGORY_CDE::VARCHAR(10)       AS CATEGORY_CDE,
    UNIT_RETAIL_AMT::NUMBER(10,2)   AS UNIT_RETAIL_AMT,
    LOADED_DT::TIMESTAMP_NTZ        AS SOURCE_LOAD_TS,
    _METADATA_FILENAME              AS SOURCE_FILE_NM
FROM {{ source('rms_raw', 'PROD_ITEM_STG') }}
WHERE ITEM_IDNT IS NOT NULL
```

#### Intermediate Model — `int_rms__prod_item_cdc.sql`

```sql
-- models/intermediate/int_rms__prod_item_cdc.sql
{{
    config(materialized='ephemeral')
}}

SELECT *
FROM (
    SELECT
        ITEM_IDNT,
        ITEM_DESC,
        CATEGORY_CDE,
        UNIT_RETAIL_AMT,
        SOURCE_LOAD_TS,
        ROW_NUMBER() OVER (
            PARTITION BY ITEM_IDNT
            ORDER BY SOURCE_LOAD_TS DESC
        ) AS ROW_RANK
    FROM {{ ref('stg_rms__prod_item') }}
)
WHERE ROW_RANK = 1
```

#### Core Dimension — `dim_prod_item.sql`

```sql
-- models/marts/core/dim_prod_item.sql
{{
    config(
        materialized='incremental',
        unique_key='PROD_ITEM_KEY',
        incremental_strategy='merge',
        on_schema_change='append_new_columns'
    )
}}

WITH source AS (
    SELECT * FROM {{ ref('int_rms__prod_item_cdc') }}
),
current_dim AS (
    SELECT * FROM {{ this }}
    WHERE CDW_RECD_CURR_FLG = 'Y'
),
changed AS (
    SELECT s.*
    FROM source s
    LEFT JOIN current_dim d ON s.ITEM_IDNT = d.ITEM_IDNT
    WHERE d.ITEM_IDNT IS NULL
       OR s.CATEGORY_CDE    <> d.CATEGORY_CDE
       OR s.UNIT_RETAIL_AMT <> d.UNIT_RETAIL_AMT
)
SELECT
    {{ dbt_utils.generate_surrogate_key(['ITEM_IDNT', 'SOURCE_LOAD_TS']) }}
        AS PROD_ITEM_KEY,
    ITEM_IDNT,
    ITEM_DESC,
    CATEGORY_CDE,
    UNIT_RETAIL_AMT,
    CURRENT_TIMESTAMP()         AS CDW_RECD_LOAD_DT,
    '9999-12-31'::TIMESTAMP_NTZ AS CDW_RECD_CLOSE_DT,
    'Y'                         AS CDW_RECD_CURR_FLG
FROM changed
```

#### Consumer View — `v_dim_prod_item_curr.sql`

```sql
-- models/marts/exposure_views/v_dim_prod_item_curr.sql
{{
    config(materialized='view', schema='OUT')
}}

SELECT
    PROD_ITEM_KEY,
    ITEM_IDNT,
    ITEM_DESC,
    CATEGORY_CDE,
    UNIT_RETAIL_AMT,
    CDW_RECD_LOAD_DT
FROM {{ ref('dim_prod_item') }}
WHERE CDW_RECD_CURR_FLG = 'Y'
```

### dbt CLI Execution Commands

```bash
# Install dependencies
dbt deps

# Run full pipeline
dbt run --profiles-dir . --target prod

# Run specific layer only
dbt run --select staging.*
dbt run --select marts.core.*
dbt run --select marts.exposure_views.*

# Run incremental only
dbt run --select fct_sales_transaction_plu_atm --vars '{"is_incremental": true}'

# Run data quality tests
dbt test --select marts.core.*

# Generate and serve documentation
dbt docs generate
dbt docs serve --port 8080

# Full CI pipeline
dbt deps && dbt run && dbt test && dbt docs generate
```

---

## 7. Full End-to-End Pipeline Demo Walkthrough

### Source Record Trace — Product Item Change Event

#### Step 1: Raw File Lands in Azure Data Lake

```
/adls/biw/raw/rms/prod_item/2026/08/26/
  → prod_item_delta_20260826_060000.parquet
```

**File contains:**
```
ITEM_IDNT=987654321, ITEM_DESC="Free Range Eggs 12pk", 
CATEGORY_CDE="DAIRY", UNIT_RETAIL_AMT=6.50 → 7.20 (PRICE CHANGE)
```

#### Step 2: COPY INTO Staging (STG)

```sql
COPY INTO STG.RMS_PROD_ITEM_STG
FROM @BIW_AZURE_STAGE/rms/prod_item/2026/08/26/
FILE_FORMAT = (FORMAT_NAME = 'BIW_PARQUET_FORMAT');
-- Result: 1 row loaded in 0.8 seconds
```

#### Step 3: CDC Merge into Persisted Staging (PSG)

```sql
-- 1 new CDC event appended to PSG.RMS_PROD_ITEM_TXN
-- ITEM_IDNT=987654321, UNIT_RETAIL_AMT=7.20, CDW_PROCESS_KEY=20260826001
```

#### Step 4: Intermediate Window Qualification (WRK)

```sql
-- QUALIFY ensures latest event per ITEM_IDNT
-- ROW_RANK=1 selected: UNIT_RETAIL_AMT=7.20
```

#### Step 5: SCD Type 2 Dimension Update (DIM_PROD_ITEM)

```sql
-- BEFORE:
-- ITEM_IDNT=987654321, UNIT_RETAIL_AMT=6.50, CDW_RECD_CURR_FLG='Y'

-- Step 5a: Expire old row
-- CDW_RECD_CURR_FLG='N', CDW_RECD_CLOSE_DT='2026-08-26 06:15:00'

-- Step 5b: Insert new active row
-- ITEM_IDNT=987654321, UNIT_RETAIL_AMT=7.20, CDW_RECD_CURR_FLG='Y'
-- CDW_RECD_LOAD_DT='2026-08-26 06:15:00'
```

#### Step 6: Point-in-Time Fact Join (FCT_SALES_TRANSACTION_PLU_ATM)

```sql
-- Transaction at 05:30 AM → joins to old price row (6.50) ✅ Correct PIT price
-- Transaction at 07:00 AM → joins to new price row (7.20) ✅ Correct PIT price
```

#### Step 7: Consumer View Presentation

```sql
-- OUT.V_DIM_PROD_ITEM_CURR returns:
-- ITEM_IDNT=987654321, UNIT_RETAIL_AMT=7.20 (current active price)
-- PowerBI dashboard reflects updated price immediately ✅
```

---

## 8. System Design Tradeoffs & Failure Modes Runbook

| Failure Scenario | Root Cause | System Impact | Mitigation Strategy |
|:----------------|:-----------|:--------------|:--------------------|
| **Late-Arriving Facts** | Sales transaction arrives before product master data is loaded | `INNER JOIN` drops revenue records silently | **Inferred Dimension Pattern** — insert placeholder row (`ITEM_DESC='Inferred'`), update when master arrives |
| **SCD Type 2 Churn** | High-frequency attribute changes (daily prices) on 100M+ row dimensions | Micro-partition churning, compute cost spikes | **Outrigger Pattern** — split rapid-change attributes into separate `PROD_ITEM_PRICE_HIST_ATM` mini-dimension |
| **Out-of-Order CDC Events** | CDC stream delivers UPDATE before INSERT | Multiple active rows (`CDW_RECD_CURR_FLG='Y'`) per business key | **Window Qualification** — `QUALIFY ROW_NUMBER() OVER (PARTITION BY ITEM_IDNT ORDER BY SOURCE_TS DESC) = 1` |
| **Unclustered Fact Explosion** | Multi-TB fact tables scanned without partition pruning | Dashboard timeouts, warehouse credit blowout | **Liquid Clustering** — `cluster_by=['TRAN_DT', 'ORG_LOC_KEY']` on all fact models |
| **Schema Evolution Drift** | Upstream adds/drops columns without notice | `SELECT *` views break, dbt compilation fails | `on_schema_change='append_new_columns'` + ban `SELECT *` in all consumer views |
| **Pipeline SLA Breach** | Upstream source delay past 4:30 AM | 6:00 AM SLA missed, Feature Store stale for forecasting | **Airflow SLA Sensors** — alert on-call team, trigger auto-retry with exponential backoff |

---

## 9. Lakehouse vs Data Warehouse — Key Concepts

### Platform Comparison

| Capability | Snowflake (Data Warehouse) | Databricks (Lakehouse) |
|:-----------|:--------------------------|:-----------------------|
| **Primary Workload** | BI, Reporting, Kimball Star Schemas | ML Feature Engineering, Model Training & Inference |
| **Core Technology** | Snowflake SQL, dbt, Virtual Warehouses | PySpark, Delta Lake, Feature Store, Unity Catalog |
| **Data Structure** | Star Schemas (`dim_*`, `fct_*`), Views | Delta Tables, Feature Tables, Parquet Streams |
| **History Tracking** | SCD Type 2 (`CDW_RECD_CURR_FLG`) | Delta Lake Time-Travel (`VERSION AS OF`) |
| **Key Output** | Conformed BI Marts (`OUT` Layer) | Point-in-Time Feature Matrices (`f_pfs_event_long_snapshot`) |
| **Query Language** | SQL | PySpark, Spark SQL, Python |
| **Governance** | Snowflake RBAC + Dynamic Data Masking | Unity Catalog + Column-Level Access Controls |

### Medallion Architecture Mapping

```
Bronze Layer  →  STG / PSG (Raw, untransformed data)
Silver Layer  →  WRK / CR  (Cleansed, deduplicated, validated)
Gold Layer    →  BIW_DWH / OUT (Conformed, business-ready)
```

### Key Terms to Use in Interviews

| Term | Definition | Your Experience |
|:-----|:-----------|:----------------|
| **Lakehouse** | Data Lake + Warehouse ACID semantics | Databricks + Delta Lake |
| **Medallion Architecture** | Bronze/Silver/Gold data quality layers | `STG` → `PSG` → `BIW_DWH` |
| **Delta Lake** | Open-source ACID storage layer on object storage | Product Feature Store |
| **Time Travel** | Query historical data state by version/timestamp | PIT feature generation |
| **Unity Catalog** | Unified Databricks governance & lineage layer | Feature Store ACL management |
| **Delta Sharing** | Open protocol for cross-platform data sharing | Snowflake ↔ Databricks bridge |
| **Lambda Architecture** | Separate batch + speed layers (largely avoided now) | Replaced by unified Delta pipelines |

---

## 10. STAR Interview Playbook — Project Atlantis

### 10.1 — 90-Second Elevator Pitch

> *"I'd love to walk you through **Project Atlantis**, which is the Smkt Product Feature Store (PFS/PFUS) & Forecasting Data Engine I recently delivered.*
>
> The core problem was that our downstream machine learning forecasting models suffered from data latency, point-in-time leakage, and inconsistent product range and price data between legacy BIW systems and our downstream forecast engines.*
>
> I led the data engineering lifecycle from business requirement discovery down to production deployment on Snowflake and Databricks. I architected a 6-tier data warehouse topology and curated a Point-in-Time feature store with automated quality validation.*
>
> As a result, we reduced pipeline data latency from daily batch to near-real-time CDC ingestion, eliminated feature leakage for model training, achieved 100% data reconciliation, and deployed production runbooks that reduced operational incidents by over 40%."*

---

### 10.2 — Requirement Gathering

**Question:** *"How do you approach requirement gathering for an enterprise data engineering project?"*

**STAR:**
- **Situation:** Forecasting teams had conflicting definitions of active product ranges, pending ranges, and promotional pricing across legacy Oracle BIW and downstream engines.
- **Task:** Translate ambiguous business requirements into formal technical specifications, data grain definitions, freshness SLAs, and target schema definitions.
- **Action:** Organized cross-functional workshops with BAs, Data Scientists, and Operations leads. Mapped data lineage from source systems (MRL/RMS). Specified TO-BE range logic for pending vs approved ranges.
- **Result:** Secured formal sign-off on HLD, eliminating mid-development scope creep.

**Spoken Script:**
> *"I anchor requirement discovery around four pillars:*
> 1. **Data Grain & Velocity** — atomic grain at Product-Item-Location-Day, with 6:00 AM daily SLA.
> 2. **Business Logic Discrepancies** — normalizing 'Active Range' differences between supply chain and merchandising into explicit flags (`IS_MERCH_RANGED_FLG`, `IS_SUPPLY_ACTIVE_FLG`).
> 3. **Point-in-Time History Needs** — SCD Type 2 requirement for ML model training without data leakage.
> 4. **Data Quality Contracts** — threshold tolerances for range reconciliation between RMS and MRL before publishing to feature tables."*

---

### 10.3 — System Architecture

**Question:** *"Walk me through the end-to-end architecture and layering of your data warehouse platform."*

**Spoken Script:**
> *"I designed a decoupled 6-tier lakehouse/data warehouse pattern in Snowflake:*
> 1. **Landing Layer (EDL Storage)** — Encrypted Parquet files from source systems via Azure Blob Storage.
> 2. **Staging Layer (STG)** — Transient tables loaded via Snowflake `COPY INTO`, truncated after execution.
> 3. **Persisted Staging (PSG)** — Retains raw CDC history with full audit columns.
> 4. **Work & Common Repository Layer (WRK/CR)** — dbt intermediate models for windowing, deduplication, surrogate key generation.
> 5. **Core DWH Layer (BIW_DWH/DM)** — Kimball Star Schema with conformed dimensions and atomic facts.
> 6. **Consumer View Layer (OUT)** — Zero-copy logical Snowflake views exposed to PowerBI, Tableau, and feature store ingestors."*

---

### 10.4 — Data Modeling & Feature Store Design

**Question:** *"Explain your data modeling approach and how you implemented history tracking and feature store delivery."*

**Spoken Script:**
> *"For data modeling, I apply strict Kimball Dimensional Standards for our analytical core, paired with specialized Point-in-Time models for feature store delivery.*
>
> Fact tables represent atomic transaction events, surrounded by conformed dimensions. All surrogate keys are deterministic hashes generated via `dbt_utils.generate_surrogate_key`.*
>
> For SCD Type 2, when an attribute changes, we expire the active row (`CDW_RECD_CURR_FLG='N'`, close timestamp set) and insert a new active row.*
>
> For the Product Feature Store (PFS), ML models require training datasets as of a specific date. To prevent data leakage, I built Point-in-Time feature event tables using timestamp range joins against SCD Type 2 dimension states."*

---

### 10.5 — Ingestion, ELT & Orchestration

**Question:** *"How do you handle ingestion, dbt transformations, and pipeline orchestration?"*

**Spoken Script:**
> *"Our ingestion pipeline leverages dbt-on-Snowflake orchestrated by Airflow and Control-M:*
> 1. **Incremental Loading** — dbt `incremental` materialization with `merge` strategy, filtered by watermark (`CDW_PROCESS_KEY > MAX(CDW_PROCESS_KEY)`).
> 2. **Out-of-Order CDC** — `QUALIFY ROW_NUMBER() OVER (PARTITION BY NATURAL_KEY ORDER BY SOURCE_TIMESTAMP DESC) = 1` in intermediate WRK layer.
> 3. **DAG Dependency Chain** — `Stage Land` → `Raw CDC Merge` → `Conformed DWH Incremental` → `Feature Snapshot` → `Data Quality Assertions` → `View Refresh`, with SLA sensors alerting if stages delay past 4:30 AM."*

---

### 10.6 — Security, Governance & Data Quality

**Question:** *"How do you handle RBAC security, data masking, and testing?"*

**Spoken Script:**
> *"Security and governance are built into the pipeline design from day one:*
>
> **Snowflake RBAC:** `BIW_TRANSFORMER_ROLE` has write access restricted to `STG`, `PSG`, `BIW_DWH`. `BIW_ANALYST_ROLE` has read-only access to `OUT` views. Dynamic Data Masking on sensitive cost attributes.*
>
> **Data Quality — three tiers:**
> - Structural: Primary key uniqueness, non-null checks, FK referential integrity.
> - Business Rules: Custom SQL assertions (non-negative quantities, valid price ranges).
> - Migration Parity: Automated reconciliation comparing row counts, checksums, and sum totals between Oracle BIW and Snowflake."*

---

### 10.7 — Deployment, Cutover & Hypercare

**Question:** *"Walk me through your production cutover and hypercare strategy."*

**Spoken Script:**
> *"Our production cutover used a Blue-Green View Repointing strategy:*
> 1. **Parallel Execution** — ran Snowflake pipelines in parallel with legacy for two full weeks, with daily reconciliation verifying zero variance.
> 2. **Zero-Downtime Repointing** — single DDL: `CREATE OR REPLACE VIEW OUT.PSG_PROD_ITEM AS SELECT * FROM BIW_DWH.DIM_PROD_ITEM`. Seamlessly switched physical source without breaking any consumer endpoint.
> 3. **Hypercare** — comprehensive operational runbooks, knowledge transfer sessions, maintained 99.98% pipeline uptime throughout."*

---

### 10.8 — Deep-Dive Technical Q&A

#### Q: How do you handle late-arriving dimension records?

> *"I implement the Inferred Dimension Member Pattern. When the fact model encounters an unknown `NATURAL_KEY`, it dynamically inserts a placeholder row with surrogate key, sets attributes to `'Inferred'`, and flags `IS_INFERRED_FLG='Y'`. When the actual master record arrives, SCD Type 1 updates overwrite the placeholder while preserving the original surrogate key connection."*

#### Q: Why Kimball over Data Vault 2.0?

> *"Data Vault offers superior agility for multi-source integration with frequent schema changes, but introduces significant join complexity that slows BI query performance. Because our primary consumers are PowerBI dashboards and feature extractors requiring fast aggregated queries, Kimball Dimensional Modeling was optimal — providing business-intuitive Star Schemas with superior query performance."*

#### Q: What is a Lakehouse and how did you implement one?

> *"A Lakehouse combines cost-efficient Data Lake storage with Data Warehouse ACID semantics. In practice, our Databricks platform uses Delta Lake — giving us versioned, ACID-compliant tables on Azure Blob Storage. This provides time-travel queries (critical for PIT feature generation), schema enforcement and evolution, and unified batch-plus-streaming in a single pipeline."*

---

## 11. Self-Introduction Scripts — Principal Data Engineer Profile

### 11.1 — 30-Second Elevator Pitch

> *"I'm a Principal Data Engineer specialising in cloud data warehouse architecture and feature store engineering. I build scalable Kimball-modeled dimensional warehouses using dbt and Snowflake, alongside PySpark and Delta Lake feature pipelines that feed Point-in-Time training and inference datasets into machine learning forecasting engines."*

### 11.2 — 60-Second Opening (Balanced Profile)

> *"I'm a Principal Data Engineer with deep expertise across both enterprise cloud data warehousing and Databricks-based feature store engineering.*
>
> On the data warehouse side, I architect and build Kimball-modeled data platforms on Snowflake — leading BIW cloud migrations, designing 6-tier schema topologies, and building incremental ELT pipelines with dbt.*
>
> On the AI/ML side, I design and operationalise Databricks Feature Stores — using PySpark and Delta Lake to engineer Point-in-Time feature tables for retail demand forecasting. This includes curating offline training features with time-travel lookup to prevent data leakage and serving online features to ML models like Relex and Advanced Analytics.*
>
> What sets my work apart is bridging these two worlds — ensuring clean, conformed master data flows seamlessly from Snowflake into Databricks Delta Lake, serving both executive BI dashboards and downstream predictive ML models with 100% data consistency."*

### 11.3 — 90-Second Deep Architecture Intro (Senior / Panel Interviews)

> *"I'm a Principal Data Engineer with extensive experience designing and scaling end-to-end data architectures — balancing Snowflake for enterprise data warehousing and Databricks for AI/ML feature store platforms.*
>
> In my recent work with supermarket retail analytics, I led two major parallel initiatives:*
>
> First, Enterprise Data Warehouse Architecture: I led the design and migration of our core BIW data warehouse onto Snowflake using Kimball Dimensional Modeling. I established a decoupled multi-tier schema topology, implemented dbt-driven incremental CDC merge models, and engineered SCD Type 2 history tracking for core dimensions.*
>
> Second, Databricks Product Feature Store (PFS/PFUS): To power ML demand forecasting for Fresh Produce and retail categories, I engineered a high-throughput Feature Store on Databricks using Delta Lake and PySpark. I built automated feature pipelines transforming supply chain, range, price, and promotional data into Point-in-Time feature tables — completely eliminating data leakage during ML backtesting.*
>
> I also architected the interoperability layer between Snowflake and Databricks — using Delta Sharing and Snowflake connectors to maintain a single source of truth across both analytical BI and predictive ML workloads."*

---

### Key Architect Phrases to Use in Every Interview

| Topic | Anchor Phrase |
|:------|:-------------|
| Architecture | *"6-tier decoupled schema topology on Snowflake"* |
| Modelling | *"Kimball Dimensional Modeling with SCD Type 2 history tracking"* |
| Engineering | *"dbt incremental CDC merge pipelines with automated data quality contracts"* |
| Feature Store | *"Point-in-Time feature snapshots for ML demand forecasting engines"* |
| Lakehouse | *"Medallion Architecture — Bronze/Silver/Gold mapped to STG/PSG/BIW_DWH"* |
| Migration | *"Zero-downtime Blue-Green view repointing cutover"* |
| Collaboration | *"Cross-functional workshops, business sign-off, and production runbooks"* |

---

*© Aneetha Chandhirasekar — Principal Data Engineer | Lead Data & Solution Architect*
*Melbourne, VIC, Australia | aneetha.chandru108@gmail.com | linkedin.com/in/aneethachandhirasekar/*
