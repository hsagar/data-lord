Below is a “single-source-of-truth” study guide that consolidates every technical detail you need for the **Databricks Academy course *Data Ingestion with Lakeflow Connect***.  
Nothing else is required—bookmark this page and you can skip the official docs while taking the course or preparing for certification.

---

# 📚 Course at a Glance
| Item | Value |
|---|---|
| **Title** | Data Ingestion with Lakeflow Connect |
| **Part of** | *Data Engineering with Databricks* learning path (Module 1 of 4) |
| **Duration** | ± 4 hours hands-on lab + self-paced videos |
| **Delivery Mode** | Databricks Academy learning portal (browser labs) |
| **Prerequisites** | • Databricks workspace basics<br>• SQL or PySpark fundamentals<br>• Conceptual knowledge of Delta Lake & Medallion architecture |
| **Learning Outcomes** | By the end you can design, build, schedule and govern ingestion pipelines from 100+ sources into Unity-Catalog-managed Delta tables using Lakeflow Connect Standard & Managed connectors, plus debug schema drift, flatten JSON, and apply incremental strategies. |

---

# 1. Lakeflow Connect Fundamentals

### 1.1 Position in Databricks Lakeflow Stack
Lakeflow is the **declarative data-engineering layer** announced April 2025.  
It has three pillars:

| Component | Purpose |
|---|---|
| **Lakeflow Connect** | **Ingestion** (this course) |
| **Lakeflow Declarative Pipelines** | Transformation (Delta Live Tables 2.0) |
| **Lakeflow Jobs** | Orchestration & scheduling |

All three run **serverless** and are natively integrated with Unity Catalog & Serverless Compute (up to 3.5× faster, 30 % cheaper) .

### 1.2 Service Models
| Option | When to Choose |
|---|---|
| **Fully-managed connectors** (UI / API) | 90 % use-cases; zero infra; auto-scale; built-in governance |
| **Custom pipelines** | Need non-supported sources or advanced transforms; use Lakeflow Declarative Pipelines or Structured Streaming |

---

# 2. Connector Taxonomy

| Category | Sub-type | Typical Sources | Key Traits |
|---|---|---|---|
| **Standard Connectors** | Batch / Incremental | Cloud object storage (S3, ADLS, GCS) | You own infra; CTAS / COPY INTO / Auto Loader |
| **Managed Connectors** | Batch / Incremental / Streaming | SaaS (Salesforce, SAP, Workday), RDBMS (SQL Server, Postgres), Pub/Sub (Kafka, Kinesis) | Databricks hosts ingestion gateway; point-and-click UI; built-in CDC or cursor-based incremental |

---

# 3. Ingestion Patterns & Syntax Recipes

### 3.1 Batch Ingestion from Object Storage
#### Pattern A – CTAS (green-field)
```sql
CREATE OR REPLACE TABLE bronze.my_table
AS SELECT *
FROM cloud_files(
      '/mnt/raw/my_data',
      'csv',
      map("header", "true")
);
```
- **Pros**: one-liner, infers schema, creates table immediately  
- **Cons**: overwrites every run → not incremental

#### Pattern B – COPY INTO (append)
```sql
COPY INTO bronze.my_table
FROM '/mnt/raw/my_data'
FILEFORMAT = CSV
FORMAT_OPTIONS ('header'='true')
COPY_OPTIONS ('mergeSchema'='true');
```
- Idempotent; appends new files only  
- Use `force=false` (default) to avoid re-processing

#### Pattern C – Auto Loader (streaming + incremental)
```python
(spark.readStream
      .format("cloudFiles")
      .option("cloudFiles.format", "json")
      .option("cloudFiles.schemaLocation", "/dbfs/mnt/checkpoint/raw_json")
      .load("/mnt/raw/json")
      .writeStream
      .option("checkpointLocation", "/dbfs/mnt/checkpoint/bronze_json")
      .table("bronze.json_events"))
```
- Detects new files via event notifications  
- Handles schema drift with `rescuedDataColumn`

---

### 3.2 Adding Metadata Columns (Bronze Enrichment)
```sql
SELECT
  *,
  _metadata.file_path  AS source_file,
  _metadata.file_modification_time AS file_time,
  current_timestamp()  AS processing_time
FROM cloud_files(...)
```

---

### 3.3 Handling Schema Drift & Rescued Data
- **Auto-generated column** `_rescued_data` (string) catches any fields not in target schema  
- **Strategies**  
  1. Inspect with `SELECT _rescued_data FROM bronze.table WHERE _rescued_data IS NOT NULL`  
  2. Evolve schema: `ALTER TABLE bronze.table ADD COLUMNS (...)`  
  3. Parse JSON semi-structured later in Silver layer using `from_json`

---

### 3.4 Flattening Semi-Structured JSON
Example nested payload:
```json
{"id":1,"user":{"name":"Ana","address":{"city":"NY"}},"tags":["A","B"]}
```
Silver transformation:
```sql
SELECT
  id,
  user.name      AS user_name,
  user.address.city AS city,
  explode(tags)  AS tag
FROM bronze.json_events;
```

---

### 3.5 Managed Connectors Walk-Through (UI Flow)
1. **New → Data Ingestion Pipeline**  
2. **Add data → Managed connector** (e.g., SQL Server)  
3. **Create Connection**  
   - Host, port, database, username, password (stored as Unity Catalog secret)  
   - Private Link / VNet peering options for on-prem   
4. **Select tables** → **Target catalog.schema**  
5. **Schedule** (cron or event-driven)  
6. **Advanced options**  
   - CDC toggle → auto-ingests only changed rows  
   - Staging location (managed ADLS/S3 bucket auto-provisioned)  
7. **Deploy via Databricks Asset Bundle** (YAML) for CI/CD 

---

# 4. Incremental Ingestion Deep Dive

| Source Mechanism | Lakeflow Connect Behavior | Config Parameters |
|---|---|---|
| **SQL Server CDC** | Reads change tables, emits only delta; supports before/after image | `enable_cdc=true` |
| **Salesforce** | Uses SystemModstamp cursor column | Auto-detected |
| **File landing** | Uses filename / modification time | Auto Loader `cloudFiles` options |

---

# 5. Alternate Ingestion Strategies Covered

| Strategy | When | How |
|---|---|---|
| **MERGE INTO** | Upserting into Silver/Gold | `MERGE INTO gold.customers c USING updates u ON c.id=u.id WHEN MATCHED THEN UPDATE SET ... WHEN NOT MATCHED THEN INSERT ...` |
| **Databricks Marketplace** | Re-use open datasets | Browse → “Get data” → auto-creates read-only Delta sharing tables |

---

# 6. Governance & Ops Checklist

| Task | Tool | How |
|---|---|---|
| **Lineage** | Unity Catalog | Automatic from ingestion → gold |
| **ACLs** | Unity Catalog | `GRANT SELECT ON TABLE bronze.raw TO group analysts;` |
| **Monitoring** | Lakeflow Jobs UI | View ingestion tasks, retries, SLAs |
| **Cost Control** | Serverless compute auto-terminates; budget alerts in Account Console |

---

# 7. Lab Exercises & Expected Artifacts

| # | Activity | What You’ll Produce |
|---|---|---|
| 1 | Setup workspace + UC catalog | `dev_catalog.raw`, `dev_catalog.staging`, `dev_catalog.curated` schemas |
| 2 | Ingest CSV → Bronze with CTAS | `bronze.transactions` |
| 3 | Switch to COPY INTO incremental | Same table, append only |
| 4 | Auto-Loader + schema drift | `bronze.logs` with `_rescued_data` |
| 5 | Managed SQL Server connector | `bronze.orders` incremental CDC |
| 6 | Flatten JSON → Silver | `silver.orders_flat` |
| 7 | Orchestrate hourly | Lakeflow Job with one task |
| 8 | Deploy via Asset Bundle | YAML pushed to Git repo |

---

# 8. Common Pitfalls & Remedies

| Symptom | Root Cause | Fix |
|---|---|---|
| “File already processed” warnings | Duplicate file names | Use `COPY INTO` with `fileNamePattern` or Auto Loader with SQS |
| `_rescued_data` exploding | Source added new columns | `ALTER TABLE ADD COLUMNS`, then re-run |
| Managed connector timeout | Firewall / no egress rule | Add Private Link or open 443 outbound |
| Cost spikes | Large initial load on serverless | Use batch window filters, or run first load on classic cluster |

---

# 9. Post-Course Next Steps

1. **Certification**: The module maps directly to the *Associate Data Engineer* exam blueprint.  
2. **Module 2**: *Deploy Workloads with Lakeflow Jobs* – learn cron, SLAs, retries.  
3. **Module 3**: *Build Data Pipelines with Lakeflow Declarative Pipelines* – transform Silver → Gold incrementally.  
4. **Module 4**: *Data Management & Governance with Unity Catalog* – master fine-grained ACLs, row/column masks.

---

# 10. Quick Reference Cheat-Sheet

```sql
-- Bronze CTAS template
CREATE OR REPLACE TABLE bronze.table
AS SELECT *, _metadata.*
FROM cloud_files('path', 'csv');

-- Incremental append template
COPY INTO bronze.table
FROM 'path'
FILEFORMAT = PARQUET;

-- Auto Loader streaming template
(spark.readStream
  .format("cloudFiles")
  .option("cloudFiles.format", "json")
  .load("path")
  .writeStream
  .option("checkpointLocation","/chk")
  .table("bronze.table"))
```

---

Keep this guide open during the labs—you will find every UI click, SQL snippet, and troubleshooting tip you need to complete the course without leaving the browser. Happy ingesting!
