# Databricks Cheat Sheet

A quick reference of essential Databricks concepts, explained in 1–2 sentences.

---

## Core Concepts

**DAB (Databricks Asset Bundles)**  
DAB is the native Databricks way to implement CI/CD: you define jobs, notebooks, workflows, permissions, and configurations as code, then bundle and promote them seamlessly from dev → test → prod.

**Unity Catalog (UC)**  
The centralized governance layer in Databricks for data and AI assets, providing fine-grained access control, lineage, and auditing across all workspaces.

**Delta Lake**  
An open-source storage layer that brings ACID transactions, schema enforcement, and time travel to data lakes, enabling reliable lakehouse architectures.

**DLT (Delta Live Tables)**  
A framework to declaratively build, manage, and monitor reliable ETL pipelines with built-in quality checks, lineage, and automated orchestration.

**Medallion Architecture (Bronze → Silver → Gold)**  
A best-practice pattern for structuring data pipelines: raw ingestion in Bronze, cleaned and conformed data in Silver, and curated business-ready data in Gold.

**Photon**  
Databricks’ native vectorized query engine built in C++ to accelerate SQL and DataFrame operations with lower latency and higher throughput.


**DBFS (Databricks File System)**  
A distributed file system mounted into Databricks clusters that allows seamless access to data in object storage.

**Repos**  
Integrated Git support inside Databricks to version-control notebooks, jobs, and workflows, supporting collaboration and CI/CD.

**Genie (Databricks AI Assistant)**  
An AI-powered assistant in Databricks to help generate, explain, and debug SQL, Python, and queries based on metadata and context.

---

## Data & Governance

**Delta Sharing**  
An open protocol to securely share live datasets across organizations and platforms, without data replication.

**Lineage**  
Automated tracking of data origins, transformations, and dependencies to improve governance, troubleshooting, and compliance.

**Row-Level Security (RLS)**  
Policy-based filtering in Unity Catalog that restricts which rows of a dataset each user can access.

**Attribute-Based Access Control (ABAC)**  
A flexible security model in Unity Catalog where access is granted based on user attributes, tags, or context.

**Data Masking**  
The ability to obfuscate sensitive fields (e.g., PII) via dynamic views or policies, ensuring privacy without breaking analytics.

---

## Compute & Execution

**Clusters**  
The compute resources (VMs) managed by Databricks to run notebooks, jobs, and pipelines, available as all-purpose (interactive) or job clusters.

**Jobs**  
Scheduled or triggered tasks that orchestrate notebooks, pipelines, and workflows in production environments.

**Workflows**  
End-to-end orchestrations in Databricks that combine jobs, tasks, dependencies, and external integrations.

**Serverless SQL Warehouses**  
Fully managed compute for SQL queries that autoscale and optimize performance without requiring manual cluster management.

**MLflow**  
An open-source platform integrated into Databricks for tracking experiments, managing models, and deploying ML pipelines.

---

## Developer Experience

**Databricks SQL**  
A SQL interface optimized for data exploration, dashboards, and BI, running on the Databricks Lakehouse.

**Notebooks**  
Interactive development environment in Databricks that supports multiple languages (SQL, Python, R, Scala) with collaborative editing.

**Magic Commands**  
Special notebook syntax (`%sql`, `%python`, etc.) to switch languages inline and mix workloads.

**Widgets**  
UI controls (dropdowns, text boxes, sliders) in notebooks to parameterize and re-run workflows easily.

**Secrets**  
Securely stored credentials (API keys, passwords) managed via Databricks Secret Scopes and injected into workflows at runtime.

---

## Productivity & Monitoring

**Databricks SQL Dashboards**  
Visual layer for creating and sharing dashboards directly on top of Lakehouse data using Databricks SQL.

**Lakehouse Monitoring**  
Built-in observability features to track data quality, pipeline health, costs, and performance across the platform.

**System Tables**  
Predefined tables in Unity Catalog that capture metadata, audit logs, usage, and monitoring information.

**Cluster Policies**  
Admin-defined templates that enforce configuration rules (e.g., autoscaling, VM types) to ensure cost and security compliance.

**REST API / CLI**  
Programmatic interfaces to automate jobs, clusters, and workflows, supporting DevOps and advanced integrations.

---
