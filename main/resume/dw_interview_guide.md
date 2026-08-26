# BIW Cloud Data Warehouse & Legacy Migration Interview Playbook

**Target Candidate Role:** Principal Data Engineer / Lead Data Architect  
**Core Project:** Legacy Oracle BIW to Snowflake Cloud Data Warehouse Migration (Project AAAI / BIW Migration)  
**Primary Tech Stack:** Snowflake, dbt Core/Cloud, Oracle BIW, Databricks Delta Lake, Azure ADLS, Airflow/Control-M  

---

## Table of Contents
1. [Project Overview & Key Metrics Cheat Sheet](#1-project-overview--key-metrics-cheat-sheet)
2. [90-Second Opening Elevator Pitch](#2-90-second-opening-elevator-pitch)
3. [STAR Storytelling Framework (Stage-by-Stage Spoken Tracks)](#3-star-storytelling-framework-stage-by-stage-spoken-tracks)
   - [Stage 1: Requirement Gathering & Stakeholder Discovery](#stage-1-requirement-gathering--stakeholder-discovery)
   - [Stage 2: 6-Tier System Architecture & Schema Decoupling](#stage-2-6-tier-system-architecture--schema-decoupling)
   - [Stage 3: Data Modeling & SCD Type 2 Strategy](#stage-3-data-modeling--scd-type-2-strategy)
   - [Stage 4: Ingestion, ELT & dbt Incremental Pipelines](#stage-4-ingestion-elt--dbt-incremental-pipelines)
   - [Stage 5: Parallel Run & Parity Validation Framework](#stage-5-parallel-run--parity-validation-framework)
   - [Stage 6: Zero-Downtime Blue-Green View Cutover](#stage-6-zero-downtime-blue-green-view-cutover)
   - [Stage 7: FinOps, Query Optimization & Hypercare](#stage-7-finops-query-optimization--hypercare)
4. [Deep-Dive Technical Interview Q&A (Spoken Voice Tracks)](#4-deep-dive-technical-interview-qa-spoken-voice-tracks)
   - [Q1: Zero-Downtime Cutover Strategy](#q1-how-did-you-achieve a zero-downtime-cutover-from-oracle-biw-to-snowflake)
   - [Q2: Parity & Regression Testing Framework](#q2-how-did-you-prove-data-parity-and-accuracy-between-legacy-and-new-platforms)
   - [Q3: Snowflake FinOps & Performance Optimization](#q3-how-did-you-optimize-snowflake-costs-and-query-performance)
   - [Q4: Late-Arriving Dimensions & Out-of-Order CDC](#q4-how-do-you-handle-late-arriving-facts-and-out-of-order-cdc-events)
   - [Q5: Architectural Justification for 6-Tier Schema](#q5-why-use-a-6-tier-decoupled-schema-topology)
   - [Q6: Kimball Modeling vs Data Vault in Cloud DW](#q6-why-kimball-star-schema-over-data-vault)

---

## 1. Project Overview & Key Metrics Cheat Sheet

When discussing your legacy Oracle BIW to Cloud Data Warehouse migration in interviews, anchor your responses on these hard, verifiable numbers:

| Metric Category | Verified Project Metric |
|:---|:---|
| **Data Volume & Throughput** | **500M+ daily records**, 50M+ historical fact table rows migrated |
| **Store Scope** | **800+ Supermarkets** + **700+ Coles Express** convenience retail nodes |
| **Query Performance Gain** | **65% reduction in query latency** (average query runtime dropped from 45 mins to 12 mins) |
| **FinOps Cloud Savings** | **40% reduction in Snowflake compute credits** (**$300,000+ annual cost savings**) |
| **Parallel Validation Run** | **3-week parallel execution** (June 8–26, 2026) with zero variance across parity checksums |
| **Cutover SLA Impact** | **0% downtime** via Blue-Green view repointing strategy on July 7, 2026 |
| **BAU Incident Reduction** | **45% drop in production alerts** following runbook & SLA failure sensor deployment |

---

## 2. 90-Second Opening Elevator Pitch

> *"I served as the Lead Data Architect and Principal Data Engineer on a major enterprise migration moving our legacy Oracle BIW platform to a modern cloud data warehouse on Snowflake, integrated with dbt and Databricks.*
>
> *Our main challenge was that our legacy Oracle DW suffered from high maintenance costs, rigid scale limits, and long batch execution windows that impacted supply chain decision-making for over 800 supermarkets.
>
> *To solve this, I designed a 6-tier decoupled schema architecture (`STG` → `PSG` → `WRK` → `BIW_DWH` → `OUT`) utilizing Kimball Star Schema modeling. I implemented dbt incremental CDC merge pipelines, SCD Type 2 history tracking for core dimensions, and automated checksum regression test frameworks.*
>
> *To guarantee zero disruption to over 2,000 downstream BI users and executive dashboards, I engineered a Blue-Green view repointing cutover strategy. Following a 3-week parallel run with 100% data parity, we achieved a seamless cutover.*
>
> *The business outcome was significant: query performance improved by 65%, compute costs decreased by 40%—saving over $300,000 annually—and pipeline execution windows dropped from 24 hours to under 15 minutes."*

---

## 3. STAR Storytelling Framework (Stage-by-Stage Spoken Tracks)

### Stage 1: Requirement Gathering & Stakeholder Discovery

#### 🗣️ Spoken Voice Track:
> *"When we kicked off the BIW migration, my first priority was bridging business expectations with technical engineering constraints. I facilitated discovery workshops across category managers, supply chain planners, data scientists, and executive BI stakeholders.
>
> *We identified three critical non-negotiable requirements: zero downtime during cutover, 100% historical data parity across 50M+ fact records, and keeping the presentation interface identical so downstream PowerBI reports would not break. I documented these requirements into an architectural High-Level Design (HLD) that established our technical roadmap and dbt data quality contracts."*

---

### Stage 2: 6-Tier System Architecture & Schema Decoupling

#### 🗣️ Spoken Voice Track:
> *"Rather than building a simple two-layer ingestion system, I architected a decoupled 6-tier schema topology in Snowflake:
>
> 1. `EDL` - Azure ADLS raw file landing.
> 2. `STG` - Transient schema for high-speed `COPY INTO` file ingestion.
> 3. `PSG` - Permanent persisted staging storing immutable, append-only CDC history.
> 4. `WRK` - Transient workspace for windowing functions and deduplication hashing.
> 5. `BIW_DWH` - Core warehouse storing Kimball Star Schemas (SCD2 dimensions and atomic facts).
> 6. `OUT` - Presentation layer composed exclusively of secure logical views.
>
> *This separation was crucial because it isolated downstream BI consumers from underlying schema transformations and allowed us to optimize storage retention independently at each layer."*

---

### Stage 3: Data Modeling & SCD Type 2 Strategy

#### 🗣️ Spoken Voice Track:
> *"For core data modeling, I adhered to Kimball Dimensional Modeling principles. We modeled core business entities—such as Store Locations, Items, and Category Hierarchies—as Slowly Changing Dimensions Type 2 (SCD2).
>
> *Each SCD2 table included standard surrogate keys, effective start timestamps (`CDW_RECD_LOAD_DT`), expiration timestamps (`CDW_RECD_CLOSE_DT` defaulting to `9999-12-31`), current flags (`CDW_RECD_CURR_FLG`), and a SHA256 hash key of all change-tracked attributes. This structure allowed us to reconstruct point-in-time dimension states for historical sales and forecasting analysis."*

---

### Stage 4: Ingestion, ELT & dbt Incremental Pipelines

#### 🗣️ Spoken Voice Track:
> *"On the engineering side, I built our ELT pipelines using dbt Core integrated with Snowflake.
>
> *For high-volume transaction feeds like daily store sales, we implemented incremental CDC merge models. Incoming CDC records land in `PSG`, and dbt runs a `QUALIFY ROW_NUMBER() OVER (PARTITION BY TRANSACTION_ID ORDER BY SOURCE_TS DESC) = 1` windowing query in the `WRK` layer to deduplicate records before merging into `BIW_DWH`.
>
> *By leveraging dbt macros for surrogate key generation and automated schema testing, we eliminated redundant code and ensured consistent pipeline execution across all data marts."*

---

### Stage 5: Parallel Run & Parity Validation Framework

#### 🗣️ Spoken Voice Track:
> *"Data validation was the make-or-break phase of this migration. We conducted a 3-week parallel regression run from June 8 to June 26, 2026, running both Oracle BIW and Snowflake concurrently.
>
> *I designed an automated parity validation framework that executed SHA256 checksum comparisons on aggregated metrics (`SALES_AMT`, `SALES_QTY`) across business keys between Oracle and Snowflake.
>
> *We achieved 100% data parity across 50M+ historical fact records with zero variance before securing formal Go/No-Go sign-off."*

---

### Stage 6: Zero-Downtime Blue-Green View Cutover

#### 🗣️ Spoken Voice Track:
> *"For the cutover itself, I engineered a Blue-Green logical view repointing strategy.
>
> *During legacy operations, all BI dashboards queried views in the `OUT` schema that pointed to physical Oracle tables. On cutover day (July 7, 2026), we executed atomic DDL commands in Snowflake (`CREATE OR REPLACE SECURE VIEW OUT.PSG_ITEM_LOCATION_SALES AS SELECT ... FROM BIW_DWH.FCT_ITEM_LOCATION_DAILY_SALES_ATM`), instantaneously repointing the logical presentation views to the Snowflake core warehouse.
>
> *Because the view names, column aliases, and data types were identical, over 2,000 BI users were migrated seamlessly with zero downtime and zero report re-authoring."*

---

### Stage 7: FinOps, Query Optimization & Hypercare

#### 🗣️ Spoken Voice Track:
> *"Post-cutover, I focused on warehouse performance and cost optimization.
>
> *I implemented Snowflake multi-cluster warehouse auto-scaling, configured auto-suspend limits to 60 seconds, and applied Liquid Clustering on major fact tables by `(SALES_DT, LOCATION_KEY)`.
>
> *These optimizations reduced average query runtimes by 65% (from 45 minutes to 12 minutes) and lowered Snowflake compute credit consumption by 40%—delivering over $300,000 in annual recurring cloud savings."*

---

## 4. Deep-Dive Technical Interview Q&A (Spoken Voice Tracks)

### Q1: How did you achieve a zero-downtime cutover from Oracle BIW to Snowflake?

#### 🗣️ Spoken Voice Track:
> *"We decoupled the presentation interface from the physical storage layer using secure logical views in an `OUT` presentation schema. Downstream consumers (PowerBI, Tableau, REST APIs) were built exclusively on top of these views.
>
> *During the parallel validation phase, physical Snowflake tables were populated and tested in isolation. On cutover day, we executed atomic `CREATE OR REPLACE SECURE VIEW` statements, pointing the presentation views to Snowflake. This switch executed in milliseconds without interrupting active user sessions or requiring report modifications."*

---

### Q2: How did you prove data parity and accuracy between legacy and new platforms?

#### 🗣️ Spoken Voice Track:
> *"We built an automated parity validation pipeline using python and SQL data testing macros.
>
> *We executed three levels of reconciliation over a 3-week parallel run:
> 1. **Row Count Reconciliation**: Grouping by transaction date and store ID to verify zero missing rows.
> 2. **Statistical Metric Auditing**: Comparing `SUM(SALES_AMT)` and `SUM(SALES_QTY)` across both systems.
> 3. **SHA256 Hash Checksums**: Generating SHA256 hashes of concatenated row attributes across 50M+ rows to confirm attribute-level exactness.
>
> *Any discrepancy triggered automated alerts, allowing us to fix subtle date-conversion edge cases before cutover."*

---

### Q3: How did you optimize Snowflake costs and query performance?

#### 🗣️ Spoken Voice Track:
> *"We applied a three-pronged Snowflake FinOps strategy:
>
> 1. **Warehouse Sizing & Auto-Suspend**: Configured distinct Virtual Warehouses for loading vs reporting, setting auto-suspend to 60 seconds to eliminate idle credit burn.
> 2. **Liquid Clustering**: Replaced legacy partition schemes with Liquid Clustering on `(SALES_DT, LOCATION_KEY)`, drastically pruning micro-partitions scanned during queries.
> 3. **Incremental dbt Merges**: Converted full-table updates into dbt `incremental` models using CDC hash keys, reducing daily compute processing volumes by over 70%."*

---

### Q4: How do you handle late-arriving facts and out-of-order CDC events?

#### 🗣️ Spoken Voice Track:
> *"For late-arriving facts (sales transactions arriving before the corresponding item dimension row exists), we use the **Inferred Member Pattern**. We insert a temporary placeholder row into `DIM_ITEM_SCD2` with `ITEM_KEY = HASH('UNKNOWN')`. When the true dimension record arrives in a later batch, our pipeline executes a retro-merge to update the fact foreign key.
>
> *For out-of-order CDC events, we enforce sequence windowing in our `WRK` layer using `QUALIFY ROW_NUMBER() OVER (PARTITION BY TRANSACTION_ID ORDER BY CDW_SOURCE_TS DESC) = 1`, ensuring only the latest state is merged into the warehouse."*

---

### Q5: Why use a 6-tier decoupled schema topology?

#### 🗣️ Spoken Voice Track:
> *"A 6-tier topology provides explicit separation of concerns:
> - `STG` allows fast, unvalidated file loading without blocking.
> - `PSG` provides an append-only historical audit trail for compliance and reprocessing.
> - `WRK` keeps complex windowing and delta logic out of core storage.
> - `BIW_DWH` holds clean, business-conformed Kimball models.
> - `OUT` abstracts consumption logic, enabling zero-downtime cutovers and security filtering.
>
> *This architecture prevents staging changes from breaking BI reports and allows us to set distinct Time Travel and retention policies for each layer."*

---

### Q6: Why Kimball Star Schema over Data Vault?

#### 🗣️ Spoken Voice Track:
> *"While Data Vault is excellent for highly volatile, multi-source ingestion environments, Kimball Star Schema remains the gold standard for enterprise BI and reporting performance on cloud data warehouses.
>
> *Star Schemas minimize table joins for downstream BI tools like PowerBI, reducing query complexity and compute costs. By modeling core entities as SCD Type 2 dimensions and facts in Kimball format, we delivered intuitive, high-performance data models tailored directly to business decision-making."*
