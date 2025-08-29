# Data Engineering & Analytics Glossary
---

A
-

**Airflow** – An open-source scheduler that lets you define, run and monitor **DAGs**.  
**Analytics Engineer** – The role that sits between DEs and BI; owns **dbt** models and business logic.  
**API (Application Programming Interface)** – A contract that lets two pieces of software talk; often the *source* in **ELT** pipelines.

B
-

**Batch Processing** – Ingest or transform data in large, scheduled chunks (e.g., nightly). Opposite of **Streaming**.  
**BigQuery / Snowflake / Redshift** – Cloud **OLAP** data warehouses optimized for analytic queries.  
**BI (Business Intelligence)** – Tools (Looker, Tableau) and practices that turn data into dashboards & reports.

C
-

**CDC (Change Data Capture)** – A pattern that streams only the rows that changed in a source table.  
**CI/CD** – Continuous Integration / Continuous Deployment; automatically test & deploy code (SQL, Python, Terraform).  
**Columnar Storage** – File layout (Parquet, ORC) that stores each column together → faster scans for analytics.

D
-

**DAG (Directed Acyclic Graph)** – A set of tasks linked in order; used by Airflow, Dagster, Prefect.  
**Data Catalog** – Central inventory of datasets: schema, owner, lineage, quality score.  
**Data Lake** – Cheap object storage (S3, GCS) that holds raw files in open formats (Parquet, JSON, Avro).  
**Data Mart** – Subset of the warehouse focused on one domain (e.g., *marketing mart*).  
**Data Mesh** – Organizational model that treats data as a product owned by domain teams.  
**Data Warehouse** – Curated, query-optimized store for analytics (often dimensional model).  
**dbt** – Open-source tool that turns SQL into versioned, testable, documented transformations.  
**Dimensional Modeling** – Design style (facts + dimensions) used in warehouses for fast analytic queries.

E
-

**ELT** – Extract → Load → Transform; raw data lands first (lake/warehouse), then SQL/dbt transforms it.  
**ETL** – Extract → Transform → Load; transforms happen *before* loading (more common in on-prem days).

F
-

**Fact Table** – The center of a star/snowflake schema; stores business events (sales, clicks).  
**Freshness SLA** – Contract that data must arrive within *N* minutes/hours of real-world event.

G
-

**Golden Dataset** – The officially blessed, quality-guaranteed table the business trusts.  
**Great Expectations** – Python framework to codify and run **data tests**.

H
-

**Hadoop** – Early open-source batch framework (HDFS + MapReduce); largely superseded by cloud warehouses & Spark.

I
-

**Idempotency** – Ability to run the same job twice without changing final state (crucial for retries).  
**Ingestion** – The “E” in ELT/ETL; moving data from source to lake/warehouse.  
**Incremental Model** – dbt/SQL logic that processes only new or changed rows since last run.

J
-

**Jinja** – Templating language used inside dbt models for loops, if-statements, macros.

K
-

**Kafka** – Distributed log for **streaming** events; often the transport layer in real-time pipelines.

L
-

**Lakehouse** – Architecture that adds warehouse-like features (ACID, schema) directly on a **data lake** (Delta, Iceberg, Hudi).  
**LookML / Looker** – Modeling layer that maps SQL tables to reusable business metrics & dashboards.

M
-

**Materialized View** – Pre-computed query result stored as a table for speed.  
**Medallion Architecture** – Bronze (raw) → Silver (clean) → Gold (business-ready) zones in the lakehouse.

N
-

**Near-Real-Time (NRT)** – Latency in seconds-minutes rather than hours.

O
-

**OLAP (Online Analytical Processing)** – Systems optimized for complex, read-heavy queries (warehouses).  
**OLTP (Online Transaction Processing)** – Systems optimized for quick, row-level writes (PostgreSQL, MySQL).  
**Orchestration** – Tooling (Airflow, Dagster, Prefect) that schedules, retries and monitors **DAGs**.

P
-

**Parquet** – Columnar file format; default choice for lake & warehouse ingestion.  
**Partitioning** – Splitting large tables by time or key to speed up reads and reduce cost.  
**Pipeline** – End-to-end flow: source → ingest → transform → serve → monitor.

Q
-

**Quality Score** – Numeric or categorical rating of dataset trustworthiness (null %, duplicates, SLA breaches).  
**Query Pushdown** – Warehouse executes filters/aggregations instead of moving raw data to BI server.

R
-

**Real-Time** – Sub-second latency; usually powered by **streaming** (Kafka → Flink → Redis).  
**Reverse ETL** – Sync warehouse tables back into SaaS tools (Salesforce, HubSpot) via Hightouch, Census.  
**Row-Level Security** – Grant access per user/role even inside the same table.

S
-

**Schema Registry** – Central store for Avro/JSON schemas so **streaming** producers & consumers stay in sync.  
**SCD (Slowly Changing Dimension)** – Technique to track history of dimension values (Type 2 = new row, Type 3 = new column).  
**Semantic Layer** – Business definitions (metrics, dimensions) shared across BI tools; dbt exposures & LookML are examples.  
**Serverless** – Cloud compute that autoscales to zero (BigQuery, Snowflake, Lambda).  
**SLO (Service-Level Objective)** – Measurable target such as “pipeline success ≥ 99 %”.  
**SQLMesh** – New open-source alternative to dbt that adds virtual data environments and versioned state.  
**Streaming** – Continuous processing of events as they occur (Kafka, Flink, Spark Structured Streaming).  
**Surrogate Key** – Meaningless integer ID (often auto-increment or hash) used in warehouse tables instead of natural keys.

T
-

**Transformation** – Turning raw data into analytics-ready shape; SQL, Python, Spark, dbt.  
**Type 2 Dimension** – See **SCD**.

U
-

**Upsert** – Insert new rows, update existing ones (MERGE statement). Critical for **incremental models**.

V
-

**Vector Database** – Specialized store for embeddings used in GenAI / semantic-search use cases.  
**Version Control** – Git for code (SQL, Python) and **Infrastructure-as-Code** (Terraform).

W
-

**Waterline** – Conceptual boundary below which data is too raw for business use (below *Silver* in **Medallion**).  
**Workflow** – Another word for **DAG** or **Pipeline**.

Y
-

**YAML** – Human-friendly markup used in dbt_project.yml, Airflow DAGs, GitHub Actions, etc.

Z
-

**Zero-ETL** – AWS marketing term for services that replicate OLTP data into warehouse without user-managed pipelines.

---
