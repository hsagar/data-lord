# Data Architecture Patterns Cheat-Sheet

> “A pattern is a reusable solution to a recurring problem in a given context.”

## 1. Storage & Compute Topologies
| Pattern | TL;DR | When to Pick | Trade-offs |
|---|---|---|---|
| **Data Warehouse** | Centralized, schema-on-write, SQL-centric OLAP store | Mature BI & reporting workloads | Expensive at petabyte scale, rigid for semi-structured data |
| **Data Lake** | Cheap object storage, schema-on-read, raw first | ML, data science, archive | Risk of “data swamp”; slower ad-hoc SQL |
| **Data Lakehouse** | Lake + ACID table format (Delta/Iceberg/Hudi) | Need warehouse performance at lake cost | Young ecosystem; still tuning small-file & vacuum ops |
| **Hybrid (Lake + Warehouse)** | Critical workloads in warehouse; exploratory in lake | Transitional orgs | Two engines → two governance planes |

## 2. Data Movement Patterns
| Pattern | TL;DR | When to Pick | Trade-offs |
|---|---|---|---|
| **ETL** | Transform before load | Strict PII masking, legacy EDW | Slower time-to-data; schema drift hurts |
| **ELT** | Load raw, transform in SQL engine | Cloud warehouses, versioned dbt | Storage cost ↑; need strong data tests |
| **CDC / Streaming** | Capture row-level changes in near-real-time | Fraud detection, IoT, NRT dashboards | Ops overhead; exactly-once semantics |
| **Reverse ETL** | Sync warehouse tables back into SaaS | Enable ops teams in CRM/Marketing | Additional infra; consistency vs. latency |

## 3. Organizational & Governance Patterns
| Pattern | TL;DR | When to Pick | Trade-offs |
|---|---|---|---|
| **Centralized (Monolith)** | Single platform team owns all data | Start-ups, small data stacks | Bottlenecks at scale; domain knowledge gap |
| **Data Mesh** | Domain-oriented, data-as-a-product | Large enterprises with ≥ 4 domains | Requires federated governance; upfront platform investment |
| **Data Fabric** | Automated metadata-driven integration | Hybrid/multi-cloud; many legacy systems | Heavy metadata tooling; still maturing |
| **Federated Query** | Query across DBs without moving data | Regulations forbid central copy | Performance hit; network latency |

## 4. Compute Processing Patterns
| Pattern | TL;DR | When to Pick | Trade-offs |
|---|---|---|---|
| **Batch** | Scheduled, high-throughput transforms | Nightly finance close, daily ML retrain | Latency T+1 |
| **Micro-Batch** | 1–5 min windows | Near-real-time KPIs | Still not second-level |
| **Streaming** | Continuous processing (Kafka, Flink) | Real-time recommendations | Hard to debug; state management |
| **Lambda** | Batch + streaming layers side-by-side | Need both historical & real-time views | Two code paths → double maintenance |
| **Kappa** | Streaming only (replay log for history) | Green-field, log-native sources | Replay cost; slower ad-hoc analytics |

## 5. Data Modeling Patterns
| Pattern | TL;DR | When to Pick | Trade-offs |
|---|---|---|---|
| **Star Schema** | Fact + dimension tables | Kimball BI, user-friendly | Denormalized storage |
| **Snowflake Schema** | Normalized dimensions | Complex hierarchies | More joins → slower |
| **Data Vault 2.0** | Hub-Sat-Link for audit & agility | Regulated industries, long history | Steep learning curve |
| **One-Big-Table (OBT)** | Flattened, wide tables | BigQuery, serverless warehouses | Duplication & cost at petabyte scale |
| **Medallion (Bronze-Silver-Gold)** | Progressive refinement in lakehouse | Standard Delta/Iceberg layout | Clear contracts, but requires compaction |

## 6. NoSQL Storage Patterns
| Pattern | TL;DR | Sweet-Spot | Trade-offs |
|---|---|---|---|
| **Key-Value Store** | Hash lookup by primary key | Caching, session store | No complex queries |
| **Document Store** | JSON/BSON blobs, nested objects | Content management, product catalog | Joins across docs are expensive |
| **Column-Family** | Columnar storage per row-key | Time-series, write-heavy | Sparse rows waste space |
| **Graph DB** | Nodes & edges with properties | Social networks, fraud rings | Sharding & global traversals hard |

---

### Quick Decision Tree
```text
Need sub-second SLAs?
├─ YES → Streaming → Lambda/Kappa
└─ NO  → Batch or Micro-Batch

Multiple domains > 3 teams?
├─ YES → Data Mesh
└─ NO  → Centralized lakehouse

Regulated & historical?
├─ YES → Data Vault or Star
└─ NO  → OBT / Medallion
```

### Keep Updated
Review this file every **6 months** or after any major infra pivot.
