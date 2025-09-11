# DLT-META – The Ultimate Reference Guide
*(Everything you need for production deployment – consolidated in one place)*

---

## 1. What is DLT-META?

DLT-META is **Databricks Labs’ open-source metadata-driven accelerator** that automates the creation and orchestration of **Bronze → Silver** [Lakeflow Declarative Pipelines](https://www.databricks.com/product/lakeflow) (formerly “Delta Live Tables”, DLT).  
Instead of writing one pipeline per data source, you describe every source, target, quality rules and transformations in a **single JSON file** (Dataflowspec). A **generic DLT pipeline** consumes this JSON and materializes:

* **Bronze layer** – raw ingestion with optional quarantine  
* **Silver layer** – cleaned, conformed, business-ready tables (SCD-1/2, append, upsert, etc.)

---

## 2. Architecture at a Glance

```text
┌────────────────────────┐
│  Dataflowspec JSON     │  ← Central metadata artifact
│  (onboarding file)     │
└────────┬───────────────┘
         │
┌────────▼───────────────┐
│  Generic DLT Pipeline  │  ← Single codebase for ALL sources
│  (Python/SQL)          │
└────┬────────────┬──────┘
     │            │
┌────▼──┐   ┌────▼──┐
│Bronze │   │Silver │
│tables │   │tables │
└───────┘   └───────┘
```

---

## 3. Core Concepts & Terminology

| Term | Meaning |
|------|---------|
| **Dataflowspec** | JSON file (or set of files) describing one or many data flows. |
| **Layer** | Either `bronze` or `silver`. |
| **Flow** | One end-to-end ingestion path: source → bronze → silver. |
| **Onboarding Job** | One-off job that writes/validates the Dataflowspec metadata. |
| **Deploy Job** | Creates/updates the actual DLT pipeline(s). |
| **Quarantine** | Separate Delta table that captures failed quality rows (Bronze). |
| **CDC** | Change-Data-Capture via `APPLY CHANGES INTO` (SCD-1/2). |

---

## 4. Dataflowspec Deep Dive

### 4.1 File Layout
```json
[
  {
    "data_flow_id": "orders_bronze_silver_flow",
    "source_details": { ... },
    "bronze_details": { ... },
    "silver_details": { ... },
    "quality_rules": [ ... ],
    "cdc": { ... }          /* optional */
  },
  ...
]
```

### 4.2 Top-Level Keys

| Key | Cardinality | Description |
|-----|-------------|-------------|
| `data_flow_id` | 1 | Unique name for the flow. |
| `source_details` | 1 | Where the data comes from. |
| `bronze_details` | 1 | How to land it raw. |
| `silver_details` | 1 | How to cleanse & model. |
| `quality_rules` | 0..n | `expect_all`, `expect_all_or_drop`, etc. |
| `cdc` | 0..1 | CDC configuration (SCD-1/2). |

### 4.3 `source_details` Object

| Attribute | Type | Values / Example |
|-----------|------|------------------|
| `source_type` | string | `cloudFiles`, `delta`, `eventhub`, `kafka`, `snapshot` |
| `source_path_or_queue` | string | `abfss://landing@datalake.dfs.core.windows.net/orders` |
| `source_format` | string | `json`, `csv`, `parquet`, `delta`, `avro` |
| `source_schema_path` | string | Optional; path to Avro/Protobuf schema file |
| `source_subscription_id` | string | Event Hub/Kafka consumer group |
| `source_connection_string_secret` | string | Databricks secret scope/key |
| `source_options` | map | Free-form Spark options (`multiLine=true`, `maxFilesPerTrigger=1000`) |

### 4.4 `bronze_details` Object

| Attribute | Type | Typical Value |
|-----------|------|---------------|
| `table_name` | string | `bronze_orders` |
| `create_table` | bool | `true` |
| `table_properties` | map | `{"pipelines.autoOptimize.zOrderCols": "order_id"}` |
| `partition_columns` | list | `["order_date"]` |
| `liquid_cluster` | bool | `true` (enables **predictive optimization**) |
| `enable_quarantine` | bool | `true` |
| `quarantine_table_name` | string | `bronze_orders_quarantine` |

### 4.5 `silver_details` Object

| Attribute | Type | Typical Value |
|-----------|------|---------------|
| `table_name` | string | `silver_orders` |
| `create_table` | bool | `true` |
| `transformation_sql_path` | string | `/Workspace/Shared/dlt-meta/transformations/orders.sql` |
| `table_properties` | map | `{"pipelines.reset.allowed": "false"}` |
| `partition_columns` | list | `["order_date"]` |
| `liquid_cluster` | bool | `true` |
| `enable_quarantine` | bool | `true` |
| `quarantine_table_name` | string | `silver_orders_quarantine` |

### 4.6 Quality Rules Array

Each element:

```json
{
  "name": "order_id_not_null",
  "expression": "order_id IS NOT NULL",
  "action": "expect_all_or_drop"     /* or expect_all / expect_all_or_fail */
}
```

### 4.7 CDC Object (optional)

```json
"cdc": {
  "primary_keys": ["order_id"],
  "sequence_by": "updated_at",
  "scd_type": 2,                    /* 1 or 2 */
  "apply_as_deletes": "op = 'D'",
  "apply_as_truncates": "op = 'T'",
  "except_column_list": ["_rescued_data"]
}
```

---

## 5. Supported Features Matrix

| Capability | Bronze | Silver | Notes |
|------------|--------|--------|-------|
| Autoloader | ✅ | ✅ | `cloudFiles` format |
| Delta | ✅ | ✅ | Streaming or batch |
| Event Hubs | ✅ | ✅ | Streaming |
| Kafka | ✅ | ✅ | Streaming |
| Snapshot | ✅ | ❌ | Bronze only |
| Custom transformations | ✅ | ✅ | Python UDFs or SQL |
| Data quality expectations | ✅ | ✅ | `expect_*` DLT syntax |
| Quarantine tables | ✅ | ✅ | Failed rows rerouted |
| Liquid clustering | ✅ | ✅ | Predictive optimization |
| SCD Type-1/2 | ✅ | ✅ | DLT apply_changes |
| Pipeline chaining | ✅ | ✅ | Single pipeline `layer=bronze_silver` |
| External sinks | ✅ | ✅ | Delta tables or Kafka |

---

## 6. Cost Model

* **DLT-META itself is free** – Apache-licensed open source.  
* **Underlying cost** = Databricks **Lakeflow Declarative Pipelines** consumption  
  – Billed via **DBUs** (serverless or classic compute).  
* **Optimization levers**  
  – Enable **Liquid clustering** (reduces re-clustering)  
  – Use **serverless DLT** pipelines (autoscale + photon)  
  – Set `maxFilesPerTrigger` to avoid small-file problem.

---

## 7. Security & Governance

| Concern | How DLT-META helps |
|---------|--------------------|
| **Secrets** | Store in Databricks secret scopes; reference in `source_connection_string_secret`. |
| **Unity Catalog** | Works out-of-the-box; target catalog/schema specified in `table_name`. |
| **Row-/Column-level security** | Use SQL UDFs in Silver transformation SQL. |
| **Audit** | All pipeline events logged to **System Tables** (`system.access.audit`). |

---

## 8. End-to-End Workflow (CLI Method)

### 8.1 One-Time Setup

```bash
# 1. Install CLI
pip install databricks-cli --upgrade
databricks auth login --host https://<workspace>.cloud.databricks.com

# 2. Install dlt-meta extension
databricks labs install dlt-meta
```

### 8.2 Onboarding Job (creates Dataflowspec)

```bash
git clone https://github.com/databrickslabs/dlt-meta.git
cd dlt-meta
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export PYTHONPATH=$(pwd)

# Interactive wizard will ask:
# - schema/database name (e.g., `demo`)
# - layer(s) to onboard (bronze, silver, or both)
databricks labs dlt-meta onboard
```
Output: Delta table `demo.dataflowspec_bronze` & `demo.dataflowspec_silver` populated.

### 8.3 Deploy the Pipelines

```bash
# Deploy Bronze pipeline
databricks labs dlt-meta deploy --layer bronze

# Deploy Silver pipeline (or combined)
databricks labs dlt-meta deploy --layer bronze_silver
```

The CLI will prompt for:
* Target catalog/schema  
* Compute type (classic or serverless)  
* Source code repo (optional)  

---

## 9. Directory Structure (Typical)

```
/workspace/
  ├── dlt-meta/
  │   ├── onboarding/
  │   │   └── dataflowspec_orders.json
  │   ├── transformations/
  │   │   └── orders.sql
  │   └── pipeline_conf/
  │       └── bronze_silver.yml
```

---

## 10. Transformation SQL Syntax

In Silver layer you author plain SQL files that can:

* Reference Bronze table as `{{ source_database }}.bronze_orders`.
* Use DLT `@dlt.table` decorators, but **do NOT** add them – the generic pipeline injects them for you.
* Example `orders.sql`:

```sql
SELECT
  order_id,
  customer_id,
  cast(order_date as date)    as order_date,
  amount::decimal(18,2)       as amount,
  _ingest_timestamp           as bronze_timestamp
FROM {{ source_database }}.bronze_orders
WHERE _rescued_data IS NULL
```

Template variables available:
* `{{ source_database }}`
* `{{ target_database }}`
* `{{ data_flow_id }}`

---

## 11. Advanced Topics

### 11.1 Pipeline Modes

| Mode | Description |
|------|-------------|
| **Direct Publishing** (default) | Pipeline publishes final tables directly. |
| **Development** | Tables suffixed with `_dev` + faster failure. |
| **Triggered** | Run once then stop. |
| **Continuous** | Streaming, 24x7. |

### 11.2 Chaining Bronze & Silver

Set `--layer bronze_silver` in deploy. The single pipeline will:

1. Ingest into Bronze
2. Wait for Bronze commits
3. Trigger Silver transformations
4. Provide atomic rollback on failure

### 11.3 UDFs & Python Functions

You can drop `.py` files under `/dlt-meta/transformations/` and reference them in SQL:

```sql
SELECT mylib.clean_amount(amount)
FROM ...
```

Ensure the wheel is installed on the cluster via `%pip install`.

### 11.4 Backfills & Idempotency

* Use DLT `pipelines.reset` option (UI or CLI) to reprocess historical data.
* Autoloader’s `cloudFiles.backfillInterval` = `'1 week'` enables automatic catch-up.

---

## 12. Monitoring & Alerting

* **DLT UI** – lineage, data quality bar charts, run history  
* **Observability metrics** – emitted to `system.lakeflow.*` Delta tables  
* **Alerting** – Databricks SQL alerts on failed expectations or pipeline SLA misses  
* **Log shipping** – Set `spark.databricks.delta.preview.enabled=true` and forward driver logs to log analytics.

---

## 13. Troubleshooting Cheat Sheet

| Symptom | Root Cause | Fix |
|---------|------------|-----|
| `Column _rescued_data is missing` | Source format mismatch | Check schema registry or add fallback columns |
| Quarantine table empty | All rows passed quality | Expected, or rules too lenient |
| Pipeline stuck in STARTING | Secrets not resolved | Verify secret scope/key spelling |
| Silver table missing CDC merge | `cdc` block malformed | Ensure primary keys & sequence_by present |
| CLI throws `ModuleNotFoundError: dlt_meta` | PYTHONPATH not set | `export PYTHONPATH=$(pwd)` inside venv |

---

## 14. Comparison Matrix (DLT-META vs. Hand-coded DLT)

| Aspect | DLT-META | Hand-coded |
|--------|----------|------------|
| Time to onboard new source | 5 min JSON entry | Hours of Python/SQL |
| Governance | Central metadata catalog | Spread across repos |
| Quality drift | Rules versioned in JSON | Manual PR reviews |
| Skill required | SQL + JSON | Python, DLT, DevOps |
| Multi-env promotion | Same JSON, diff workspace | Per-env code changes |

---

## 15. Frequently Asked Questions (Consolidated)

**Q1. Can I use Unity Catalog?**  
Yes – specify three-part table names `catalog.schema.table` in `table_name`.

**Q2. Serverless or Classic compute?**  
DLT-META works with both. Pass `--compute serverless` to CLI.

**Q3. Incremental schema evolution?**  
Leverage Autoloader’s `cloudFiles.schemaEvolutionMode = "addNewColumns"` in `source_options`.

**Q4. Can I mix streaming and batch?**  
Yes – set `source_type = "delta"` and add `trigger(availableNow=True)` option.

**Q5. Formal support?**  
No SLA from Databricks. Use [GitHub Issues](https://github.com/databrickslabs/dlt-meta/issues) for bugs or feature requests.

---

## 16. Production Checklist ✅

* [ ] Dataflowspec JSON under version control (Git).  
* [ ] Separate onboarding & deploy pipelines per environment (dev / staging / prod).  
* [ ] Grant `USE CATALOG`, `CREATE TABLE`, `MODIFY` to the DLT service principal.  
* [ ] Enable **predictive optimization** (`liquid_cluster = true`).  
* [ ] Store secrets in **Unity Catalog secret scopes**.  
* [ ] Add **data quality alerts** on quarantine row count > 0.  
* [ ] Schedule daily backfill job with `pipelines.reset = true` if needed.  

---

## 17. Quick Copy-Paste Starter

```json
[
  {
    "data_flow_id": "starter_flow",
    "source_details": {
      "source_type": "cloudFiles",
      "source_format": "json",
      "source_path_or_queue": "abfss://landing@datalake.dfs.core.windows.net/events/",
      "source_options": {
        "cloudFiles.inferColumnTypes": "true",
        "cloudFiles.schemaEvolutionMode": "addNewColumns"
      }
    },
    "bronze_details": {
      "table_name": "demo.bronze_events",
      "create_table": true,
      "enable_quarantine": true,
      "quarantine_table_name": "demo.bronze_events_quarantine",
      "liquid_cluster": true
    },
    "silver_details": {
      "table_name": "demo.silver_events",
      "create_table": true,
      "transformation_sql_path": "/Workspace/Shared/dlt-meta/transformations/events.sql",
      "liquid_cluster": true
    },
    "quality_rules": [
      {"name": "id_not_null", "expression": "event_id IS NOT NULL", "action": "expect_all_or_drop"},
      {"name": "valid_type", "expression": "event_type IN ('click','view','purchase')", "action": "expect_all"}
    ]
  }
]
```

---

You now have **every detail required to run DLT-META in production without referring to any external documentation**. Happy building!
