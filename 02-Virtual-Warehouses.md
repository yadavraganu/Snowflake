# Snowflake Virtual Warehouses
<br></br>
## 1. What is a Virtual Warehouse?

A **virtual warehouse** is a named, user-managed **MPP compute cluster** that executes queries, DML, and data loading in Snowflake. It provides CPU, memory, and local SSD cache, but **stores no table data permanently** (data lives in the centralized storage layer).

* Created, resized, suspended, and resumed with SQL (or UI) in seconds.
* Billed in **credits**, only while running.
* Multiple warehouses can read the same data at the same time with **no compute contention** between them.
* A query runs on **one warehouse** (or one cluster of a multi-cluster warehouse); it is not split across warehouses.

```
 Query ──► Cloud Services (parse, optimize, plan)
                │
                ▼
        ┌───────────────────────────────┐
        │   VIRTUAL WAREHOUSE (cluster) │
        │  ┌──────┐ ┌──────┐ ┌──────┐   │
        │  │ Node │ │ Node │ │ Node │   │   <- CPU + RAM + local SSD cache
        │  └──────┘ └──────┘ └──────┘   │
        └───────────────┬───────────────┘
                        ▼
            Centralized storage (micro-partitions)
```

<br></br>

## 2. Warehouse Types

| Type | Key Idea | Best For | Notes |
| --- | --- | --- | --- |
| **Standard (Gen 1)** | The original general-purpose warehouse | BI, ad-hoc SQL, ELT/ETL, most workloads | Configured with `GENERATION = '1'` / `STANDARD_GEN_1` |
| **Standard (Gen 2)** | Next generation of standard warehouses on upgraded hardware and engine improvements | Same as Gen 1, especially analytics and DML-heavy pipelines | `GENERATION = '2'` / `STANDARD_GEN_2`. Credit rate per hour differs from Gen 1, so compare cost per workload, not only speed. Availability varies by region/cloud; defaults are shifting as Gen 2 rolls out. |
| **Snowpark-optimized** | Much more memory per node (and a different local cache profile) | ML training, large in-memory processing, memory-hungry Snowpark / Python UDFs and stored procedures | `WAREHOUSE_TYPE = 'SNOWPARK-OPTIMIZED'` with `RESOURCE_CONSTRAINT = MEMORY_1X / MEMORY_16X / MEMORY_64X`. Higher credit rate than standard (roughly 1.5x at the same size). Larger memory constraints require a minimum size (e.g., Medium or larger for 16X). Not available as Gen 2. |
| **Interactive** | Built for low-latency, high-concurrency, repeatedly accessed data | Customer-facing dashboards, data apps, APIs needing sub-second responses | Used with **interactive tables**. Created with `CREATE INTERACTIVE WAREHOUSE`. Has its own constraints (query types, tables supported), so verify against current docs before choosing it. |

**Related but different:** *Snowpark Container Services* uses **Compute Pools**, not virtual warehouses. *Serverless features* (Snowpipe, Automatic Clustering, serverless Tasks, Search Optimization, etc.) use Snowflake-managed compute with no user-sized warehouse.

**Configuration dimensions (not separate types):**

* **Size** (XS to 6XL): how much compute per cluster.
* **Single-cluster vs Multi-cluster:** how many clusters for concurrency.
* **Gen 1 vs Gen 2** and **memory constraint** (for Snowpark-optimized).

<br></br>

## 3. Warehouse Sizes & Credit Consumption

Each size up **doubles** compute resources and the credit rate.

| Size | Standard Credits / Hour | Snowpark-optimized (approx.) |
| --- | --- | --- |
| X-Small | 1 | n/a or 1.5x |
| Small | 2 | n/a or 1.5x |
| Medium | 4 | ~6 |
| Large | 8 | ~12 |
| X-Large | 16 | ~24 |
| 2X-Large | 32 | ~48 |
| 3X-Large | 64 | ~96 |
| 4X-Large | 128 | ~192 |
| 5X-Large | 256 | ~384 |
| 6X-Large | 512 | ~768 |

*Gen 2 standard warehouses use a different (higher) credit rate than Gen 1; always check Snowflake's current Service Consumption Table for your cloud and region.*

**Billing rules to remember**

* Billed **per second**, with a **60-second minimum** each time a warehouse starts or resumes.
* A **suspended** warehouse costs **0** compute credits (storage is billed separately).
* **Multi-cluster:** billed per running cluster (credits/hour x number of running clusters).
* **Resizing up:** the new nodes are provisioned and billed as soon as they are added; **running queries are not affected**, only new queries use the larger size.
* **Resizing down:** removed nodes (and their local cache) go away.
* Credit price ($ per credit) depends on edition, cloud, and region. Cloud Services usage is billed only above 10% of daily warehouse credits.

**Linear scaling insight:** doubling the size roughly halves runtime for a well-parallelised query, so cost stays about the same but results arrive faster. This only holds while the query is large enough to use the extra nodes. Small queries on a big warehouse just waste credits.

<br></br>

## 4. Warehouse Lifecycle & States

```
 CREATE ──► STARTED ◄──────── auto-resume / ALTER ... RESUME
              │  ▲
   idle for   │  │ query arrives (AUTO_RESUME = TRUE)
 AUTO_SUSPEND │  │
              ▼  │
          SUSPENDED   (no credits, local cache dropped)
```

| State | Meaning |
| --- | --- |
| **Started / Running** | Compute provisioned; billing active |
| **Suspended** | No compute; no credits; local cache is cleared |
| **Resizing** | Nodes being added/removed; new queries wait for new capacity as required |
| **Resuming / Quiescing** | Starting up, or draining running queries before shutting down |

* **Auto-suspend:** Suspends after N seconds of inactivity (`AUTO_SUSPEND`). `0` / `NULL` means never suspend (usually a cost mistake).
* **Auto-resume:** `AUTO_RESUME = TRUE` starts the warehouse when a query is submitted to it (typically within seconds).
* **`INITIALLY_SUSPENDED = TRUE`:** Creates the warehouse without starting (and billing) it.
* **Cache trade-off:** Suspending clears the local SSD cache. Very aggressive auto-suspend saves credits but makes the next queries read from remote storage (slower, cold cache).

<br></br>

## 5. When a Warehouse Is Used (and When It Is Not)

A warehouse is only needed when **data has to be processed**: scanned, filtered, joined, aggregated, transformed, written, or loaded. Anything that only touches **metadata, definitions, permissions, or cached results** is handled by the **Cloud Services layer**, which needs no running warehouse and consumes no warehouse credits.

```
 Statement arrives
        │
        ▼
 CLOUD SERVICES (always on): authenticate, authorize, parse, compile, optimize
        │
        ├── Can it be answered from METADATA or RESULT CACHE, or is it a
        │   DDL / SHOW / GRANT / session command?
        │        YES ──► handled by Cloud Services, NO warehouse credits
        │
        └── Does it need to read or write DATA (micro-partitions)?
                 YES ──► runs on a VIRTUAL WAREHOUSE (credits consumed)
```

Note that **every** statement is compiled by Cloud Services first. Only the **execution** step moves to the warehouse when data processing is required.

### 5.1 Warehouse IS Used

| Operation | Why a Warehouse Is Needed |
| --- | --- |
| `SELECT` that scans table data (filters, joins, aggregations, sorting, window functions) | Reads and processes micro-partitions |
| `SELECT` on views, secure views, external tables, or shared (consumer-side) data | Same: data must be read and processed by the **consumer's** warehouse |
| Time Travel queries (`SELECT ... AT / BEFORE`) | Still a data read |
| `INSERT`, `UPDATE`, `DELETE`, `MERGE` | Reads, rewrites, and writes micro-partitions |
| `CREATE TABLE AS SELECT` (CTAS), `INSERT ... SELECT` | Executes the query and writes the result |
| `COPY INTO <table>` (bulk load from stage) | Parses and loads files into micro-partitions |
| `COPY INTO <location>` (unload) | Reads data and writes files to a stage |
| Querying staged files (`SELECT $1 FROM @stage`) | Reads and parses file contents |
| Querying a stream, or reading `ACCOUNT_USAGE` / `INFORMATION_SCHEMA` views | These are `SELECT` statements, so they need a warehouse (a small one is enough) |
| Stored procedures, UDFs, and Snowpark DataFrame actions that run queries | The queries inside them run on the warehouse |
| Dynamic table refreshes | Refresh runs on the warehouse specified for the table |
| User-managed Tasks (tasks created with a `WAREHOUSE` setting) | Task body runs on that warehouse |
| Alerts (when created with a warehouse) | Condition query runs on that warehouse |

### 5.2 Warehouse Is NOT Used (Cloud Services Only)

| Operation | Why No Warehouse Is Needed |
| --- | --- |
| **Result cache hits** (same query text, data unchanged, same role access, deterministic functions, within the cache window) | Final result is stored in Cloud Services and returned directly; no compute credits, and a suspended warehouse typically stays suspended |
| **Metadata-only queries**: e.g., `SELECT COUNT(*)`, `MIN()`, `MAX()` on a full table | Answered from micro-partition metadata (row counts, min/max) |
| **DDL**: `CREATE / ALTER / DROP` of databases, schemas, tables, views, stages, file formats, sequences, etc. | Only changes metadata and object definitions |
| `SHOW`, `DESCRIBE` commands | Metadata reads |
| **Zero-copy cloning** (`CREATE TABLE ... CLONE`) and `UNDROP` | Creates new metadata pointers to existing micro-partitions; no data is copied |
| **Access control** statements: `GRANT`, `REVOKE`, role and user management | Metadata and security operations |
| **Session / context** commands: `USE`, `SET`, `ALTER SESSION` | Control-plane state |
| **File transfer commands** `PUT` / `GET` | Executed by the client driver against internal stage storage |
| Stage metadata commands such as `LIST` and `REMOVE` (generally) | Operate on stage file listings, not table data |
| Transaction control: `BEGIN`, `COMMIT`, `ROLLBACK` | Metadata and snapshot management |
| Authentication, query parsing, optimization and compilation | Always performed by Cloud Services, even for queries that later run on a warehouse |

### 5.3 Snowflake-Managed (Serverless) Compute: Neither a Warehouse Nor "Free"

Some work does use real compute but **not your warehouse**. It runs on Snowflake-managed resources and is billed separately as **serverless credits**:

| Feature | Notes |
| --- | --- |
| **Snowpipe** / Snowpipe Streaming | Continuous file / row ingestion |
| **Automatic Clustering** | Background re-clustering of micro-partitions |
| **Materialized view maintenance** | Keeps materialized views current |
| **Search Optimization Service** maintenance | Maintains search access paths |
| **Serverless Tasks** | Tasks created without a user-managed warehouse |
| **Query Acceleration Service** | Offloads parts of eligible queries from a warehouse |
| **Replication / failover** refresh | Cross-region and cross-cloud copies |

### 5.4 Edge Cases and Gotchas

* **Warehouse selected but suspended:** Statements served by Cloud Services (cache hits, DDL, `SHOW`) do not resume it. A data-processing statement with `AUTO_RESUME = TRUE` will.
* **Running is not the same as busy:** A running warehouse is billed even when idle. Idle time before `AUTO_SUSPEND` kicks in is the most common source of waste.
* **Keep-alive and polling queries:** BI tools, monitoring scripts, or health checks that run a data-touching `SELECT` every few minutes keep a warehouse awake and billing. Use checks that don't touch table data, or point them at a dedicated X-Small warehouse, and confirm in `WAREHOUSE_EVENTS_HISTORY` that they are not triggering resumes.
* **Cloud Services is not unlimited:** Heavy metadata activity (thousands of tiny DDL/`SHOW` calls, huge numbers of small queries) consumes Cloud Services credits, billed only above the 10% daily threshold.
* **Shared data:** The **provider** needs no warehouse for consumers to query shared data, but each **consumer** pays for its own warehouse to run those queries.
* **Cloning is cheap, using the clone is not:** The clone itself needs no warehouse, but querying or modifying it does.
* **Verify with Query History:** Compare `WAREHOUSE_NAME`, execution time, and bytes scanned in `QUERY_HISTORY` to see whether a statement actually did warehouse work. Result-cache hits show near-zero execution time and no data scanned. Behaviour for edge-case statements can differ, so confirm on your own account.

### 5.5 Quick Decision Table

| Question | Answer |
| --- | --- |
| Does it read or write table data? | **Warehouse** |
| Does it only change or read object definitions and metadata? | **No warehouse** |
| Is it a repeated identical query with unchanged data? | **Likely no warehouse** (result cache) |
| Is it a plain `COUNT(*)` / `MIN` / `MAX` on a whole table? | **Likely no warehouse** (metadata) |
| Is it continuous ingestion via Snowpipe or background maintenance (clustering, MV)? | **Serverless**, not your warehouse |
| Is it just moving files with `PUT` / `GET`? | **No warehouse** |

<br></br>

## 6. Multi-Cluster Warehouses (Scale-Out for Concurrency)

A multi-cluster warehouse is **one warehouse made of 1 to 10 identical clusters** (all of the same size). It solves **concurrency / queuing**, not single-query slowness. Requires **Enterprise edition or above**.

| Mode | Setting | Behaviour |
| --- | --- | --- |
| **Maximized** | `MIN_CLUSTER_COUNT = MAX_CLUSTER_COUNT` (> 1) | All clusters start together and stay running while the warehouse is running. Best for predictable, constant high concurrency. |
| **Auto-scale** | `MIN_CLUSTER_COUNT < MAX_CLUSTER_COUNT` | Snowflake adds clusters when queries queue and removes them when load drops. Best for fluctuating concurrency. |

**Scaling policies (Auto-scale mode)**

| Policy | Goal | Behaviour |
| --- | --- | --- |
| **Standard** (default) | Prevent / minimize queuing | Starts an extra cluster as soon as queuing is detected; shuts clusters down after sustained low load |
| **Economy** | Conserve credits | Starts a new cluster only when there is enough sustained load to keep it busy for several minutes; accepts some queuing |

**Scale-up vs scale-out at a glance**

```
 SCALE UP (bigger size)              SCALE OUT (more clusters)
 ┌──────────────────┐                ┌────┐ ┌────┐ ┌────┐
 │   Larger nodes   │                │ C1 │ │ C2 │ │ C3 │
 │  one big cluster │                └────┘ └────┘ └────┘
 └──────────────────┘                 same size, more of them
 Fixes: slow big queries,             Fixes: queuing, many concurrent
        memory spilling                      users / BI dashboards
```

<br></br>

## 7. Key Parameters Reference

| Parameter | Purpose | Notes |
| --- | --- | --- |
| `WAREHOUSE_SIZE` | Compute size per cluster | XSMALL to X6LARGE |
| `WAREHOUSE_TYPE` | `STANDARD` or `SNOWPARK-OPTIMIZED` | |
| `RESOURCE_CONSTRAINT` | Memory profile / generation selection | e.g., `MEMORY_16X`, `STANDARD_GEN_2` |
| `GENERATION` | `'1'` or `'2'` for standard warehouses | Not for Snowpark-optimized |
| `AUTO_SUSPEND` | Idle seconds before suspend | Typical: 60 to 300 s; tune against cache needs |
| `AUTO_RESUME` | Start on demand | Normally `TRUE` |
| `INITIALLY_SUSPENDED` | Create in suspended state | Avoids immediate billing |
| `MIN_CLUSTER_COUNT` / `MAX_CLUSTER_COUNT` | Multi-cluster range | 1 to 10 |
| `SCALING_POLICY` | `STANDARD` or `ECONOMY` | Auto-scale mode only |
| `MAX_CONCURRENCY_LEVEL` | Target concurrent statements per cluster (default 8) | Lower value gives each query more resources |
| `STATEMENT_TIMEOUT_IN_SECONDS` | Cancel statements running too long | Guard against runaway queries |
| `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS` | Cancel statements that queue too long | |
| `ENABLE_QUERY_ACCELERATION` | Turn on Query Acceleration Service | See section 10 |
| `QUERY_ACCELERATION_MAX_SCALE_FACTOR` | Cap on extra serverless compute | `0` = unlimited within limits |
| `RESOURCE_MONITOR` | Attach a credit quota monitor | See section 12 |
| `COMMENT` | Describe owner / purpose | Useful for chargeback |

**Example DDL**

```sql
-- Standard warehouse for BI, scales out for concurrency
CREATE WAREHOUSE bi_wh
  WAREHOUSE_SIZE      = 'MEDIUM'
  AUTO_SUSPEND        = 120
  AUTO_RESUME         = TRUE
  INITIALLY_SUSPENDED = TRUE
  MIN_CLUSTER_COUNT   = 1
  MAX_CLUSTER_COUNT   = 4
  SCALING_POLICY      = 'STANDARD'
  COMMENT             = 'BI dashboards - auto-scales 1 to 4 clusters';

-- ETL warehouse, single cluster, short idle time
CREATE WAREHOUSE etl_wh
  WAREHOUSE_SIZE = 'LARGE'
  AUTO_SUSPEND   = 60
  AUTO_RESUME    = TRUE
  INITIALLY_SUSPENDED = TRUE;

-- Snowpark-optimized warehouse for memory-heavy work
CREATE WAREHOUSE ml_wh
  WAREHOUSE_SIZE      = 'MEDIUM'
  WAREHOUSE_TYPE      = 'SNOWPARK-OPTIMIZED'
  RESOURCE_CONSTRAINT = 'MEMORY_16X';

-- Operate a warehouse
ALTER WAREHOUSE etl_wh SET WAREHOUSE_SIZE = 'XLARGE';  -- affects new queries
ALTER WAREHOUSE etl_wh SUSPEND;
ALTER WAREHOUSE etl_wh RESUME IF SUSPENDED;
USE WAREHOUSE bi_wh;                                    -- session level
```

<br></br>

## 8. Concurrency, Queuing & Spilling

* **Queuing:** When a cluster has no spare resources, new queries wait. Symptoms: queued time in query history, `WAREHOUSE_LOAD_HISTORY` showing queued load. **Fix:** multi-cluster (scale out), split workloads across warehouses, or lower `MAX_CONCURRENCY_LEVEL` only if the queries are heavy.
* **Spilling:** When a query's working set exceeds memory, it spills to **local SSD**, then to **remote storage** (much slower). Symptoms: "Bytes spilled to local / remote storage" in Query Profile. **Fix:** scale up (more memory), use Snowpark-optimized for extreme cases, reduce data volume earlier in the query, or rewrite the join/aggregation.
* **Cold cache:** First queries after resume read from remote storage; keep warehouses warm for latency-sensitive workloads.
* **Partition scanning:** Poor pruning scans too many micro-partitions; the fix is data clustering / filters, not a bigger warehouse.

**Diagnosis cheat sheet**

| Symptom | Likely Cause | First Action |
| --- | --- | --- |
| Many queries "queued" | Too much concurrency | Multi-cluster / separate warehouses |
| One big query slow, heavy spilling | Not enough memory | Scale up (or Snowpark-optimized) |
| Query slow, little spilling, scans most partitions | Poor pruning | Clustering key / better filters |
| First query after idle is slow | Cold cache / resume time | Tune auto-suspend, keep warm |
| Bill high, utilization low | Oversized or never-suspending warehouse | Downsize, enable auto-suspend |
| Small query on a huge warehouse | Over-provisioned | Right-size; use a smaller warehouse |

<br></br>

## 9. Caching and Warehouses

| Cache | Tied to Warehouse? | Effect on Warehouse Design |
| --- | --- | --- |
| **Result cache** | No (Cloud Services) | Identical repeated queries return instantly without a running warehouse (24 h, extended on reuse) |
| **Metadata cache** | No (Cloud Services) | `COUNT(*)`, `MIN/MAX` can be answered without a warehouse |
| **Local disk cache** | **Yes** (per warehouse) | Lost on suspend; shared by queries on the same warehouse; **route similar queries to the same warehouse** to benefit |

**Design implication:** Dedicating a warehouse to a repeated workload (e.g., one dashboard group) improves cache hit rates; mixing unrelated workloads thrashes the cache.

<br></br>

## 10. Query Acceleration Service (QAS)

* Offloads **parts** of eligible, scan-heavy queries to Snowflake-managed **serverless compute**, so a warehouse can handle occasional large outliers without being permanently upsized.
* Useful for: ad-hoc queries with large scans and selective filters, workloads with unpredictable outlier queries.
* Controlled by `ENABLE_QUERY_ACCELERATION = TRUE` and `QUERY_ACCELERATION_MAX_SCALE_FACTOR`.
* Billed separately as serverless credits.
* Check eligibility with `SYSTEM$ESTIMATE_QUERY_ACCELERATION('<query_id>')` or the `QUERY_ACCELERATION_ELIGIBLE` view.

<br></br>

## 11. Workload Design & Sizing Best Practices

**Separate warehouses by workload** (isolation + chargeback + right-sizing):

| Workload | Typical Setup |
| --- | --- |
| Data loading / ingestion | Dedicated warehouse; small to medium; short auto-suspend |
| Transformations (ELT, dbt) | Medium to large single cluster; scale up for big batch windows |
| BI / dashboards | Medium, multi-cluster auto-scale; moderate auto-suspend to keep cache warm |
| Ad-hoc / data science | Separate warehouse per team; resource monitor attached |
| ML / memory-heavy Snowpark | Snowpark-optimized |
| Customer-facing / low-latency apps | Interactive warehouse (if workload fits) or a warm, right-sized standard warehouse |

**Sizing approach**

1. Start **small** (e.g., X-Small / Small) and measure; sizing up is instant.
2. Scale up while runtime keeps dropping roughly in proportion to size. Stop when it stops improving.
3. Use multi-cluster only for **concurrency**, not for making single queries faster.
4. Test with representative queries on a **clean cache** vs **warm cache**.
5. Revisit regularly using usage history (workloads change).

**Do**

* Set `AUTO_SUSPEND` on every warehouse; use `INITIALLY_SUSPENDED = TRUE`.
* Set `STATEMENT_TIMEOUT_IN_SECONDS` to stop runaway queries.
* Attach resource monitors to non-production and ad-hoc warehouses.
* Use clear naming, tags, and `COMMENT` (team, workload, owner).
* Keep one workload type per warehouse.

**Don't**

* Don't use one giant warehouse for everything.
* Don't scale up to fix poor pruning or bad SQL.
* Don't disable auto-suspend "to keep cache warm" without checking the cost.
* Don't use multi-cluster to speed up a single slow query.
* Don't create many tiny, mostly-idle warehouses: each resume has a 60-second minimum charge.

<br></br>

## 12. Cost Governance & Monitoring

### Resource Monitors

Credit-quota guardrails on an account or on specific warehouses.

```sql
CREATE RESOURCE MONITOR adhoc_monitor
  WITH CREDIT_QUOTA = 100
       FREQUENCY    = MONTHLY
       START_TIMESTAMP = IMMEDIATELY
  TRIGGERS ON 75  PERCENT DO NOTIFY
           ON 100 PERCENT DO SUSPEND            -- lets running queries finish
           ON 110 PERCENT DO SUSPEND_IMMEDIATE; -- cancels running queries

ALTER WAREHOUSE adhoc_wh SET RESOURCE_MONITOR = adhoc_monitor;
```

* Actions: **NOTIFY**, **SUSPEND** (after running queries finish), **SUSPEND_IMMEDIATE** (kills running queries).
* Monitors track **warehouse compute credits** (not serverless features).
* Budgets and tags complement resource monitors for broader cost control.

### Useful Monitoring Views (`SNOWFLAKE.ACCOUNT_USAGE`)

| View | Use |
| --- | --- |
| `WAREHOUSE_METERING_HISTORY` | Credits used per warehouse per hour |
| `WAREHOUSE_LOAD_HISTORY` | Running / queued / blocked load; spot queuing |
| `WAREHOUSE_EVENTS_HISTORY` | Suspend, resume, resize, cluster events |
| `QUERY_HISTORY` | Per-query runtime, queue time, bytes scanned, spilling, warehouse used |
| `QUERY_ATTRIBUTION_HISTORY` | Compute credits attributed to individual queries |
| `QUERY_ACCELERATION_ELIGIBLE` | Queries that QAS could speed up |

```sql
-- Credits by warehouse, last 7 days
SELECT warehouse_name, SUM(credits_used) AS credits
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
GROUP BY warehouse_name
ORDER BY credits DESC;

-- Queries that spilled to remote storage or queued heavily
SELECT query_id, warehouse_name, warehouse_size,
       total_elapsed_time/1000 AS elapsed_s,
       queued_overload_time/1000 AS queued_s,
       bytes_spilled_to_local_storage,
       bytes_spilled_to_remote_storage
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -1, CURRENT_TIMESTAMP())
  AND (bytes_spilled_to_remote_storage > 0 OR queued_overload_time > 0)
ORDER BY total_elapsed_time DESC
LIMIT 50;
```

*`ACCOUNT_USAGE` views have some latency (up to a few hours); `INFORMATION_SCHEMA` table functions are near real time but have shorter history.*

<br></br>

## 13. Access Control for Warehouses

| Privilege | Allows |
| --- | --- |
| `USAGE` | Run queries on the warehouse |
| `OPERATE` | Start, stop, suspend, resume, abort queries |
| `MODIFY` | Change properties (size, auto-suspend, clusters, etc.) |
| `MONITOR` | View usage and queries on the warehouse |
| `OWNERSHIP` | Full control, including grants and drop |

```sql
GRANT USAGE   ON WAREHOUSE bi_wh TO ROLE analyst;
GRANT OPERATE ON WAREHOUSE bi_wh TO ROLE data_ops;
GRANT MODIFY  ON WAREHOUSE bi_wh TO ROLE platform_admin;
```

Tip: Grant `USAGE` to many roles, but restrict `MODIFY` tightly, since resizing is the most common cause of surprise bills.

<br></br>

## 14. Common Pitfalls

* Warehouse left running with `AUTO_SUSPEND = 0` or with a very long idle timeout.
* Using a Large/XL warehouse for tiny or trivial queries.
* Treating multi-cluster as a way to make a single query faster.
* Ignoring spilling and just re-running on a bigger warehouse blindly.
* Mixing ETL and BI on one warehouse (queries starve each other).
* Forgetting the 60-second minimum when starting and stopping warehouses repeatedly.
* Assuming a resize affects queries that are already running.
* Assuming Gen 2 or Snowpark-optimized is automatically cheaper or faster: benchmark on your own workload.

<br></br>

## 15. Quick Interview Q&A

| Question | Short Answer |
| --- | --- |
| What is a virtual warehouse? | A named MPP compute cluster that runs queries; stateless with respect to table data; billed per second while running. |
| Scale up vs scale out? | Up = bigger size, for complex queries and spilling. Out = more clusters (multi-cluster), for concurrency and queuing. |
| What warehouse types exist? | Standard (Gen 1 / Gen 2), Snowpark-optimized, and Interactive; multi-cluster is a configuration of a standard warehouse. |
| Why would you use Snowpark-optimized? | Memory-heavy workloads such as ML training and large Python UDFs. |
| How is a warehouse billed? | Credits per hour by size, per second, 60-second minimum per resume; per cluster for multi-cluster; zero when suspended. |
| What happens to the cache on suspend? | The local disk cache is dropped; result and metadata caches are unaffected. |
| Does resizing affect running queries? | No, only queries submitted after the resize. |
| Maximized vs Auto-scale? | Maximized keeps a fixed number of clusters running; Auto-scale adds and removes clusters with load. |
| Standard vs Economy scaling policy? | Standard avoids queuing by starting clusters quickly; Economy saves credits and tolerates queuing. |
| How do you reduce warehouse cost? | Auto-suspend, right-size, separate workloads, resource monitors, statement timeouts, review usage history. |
| How to detect that a warehouse is too small? | Memory spilling (especially remote), long runtimes that shrink with size increases. |
| How to detect it needs more clusters? | Queued time and queued load in `QUERY_HISTORY` / `WAREHOUSE_LOAD_HISTORY`. |
| Can two warehouses write to the same table? | Yes; compute does not conflict, though `UPDATE/DELETE/MERGE` on the same table are serialized by table locks. |
| Is the Query Acceleration Service a warehouse type? | No; it is serverless compute that assists eligible queries from a normal warehouse. |
| Does every query need a running warehouse? | No. DDL, `SHOW`, grants, cloning, result-cache hits, and metadata-only queries are handled by Cloud Services. Queries that read or write data need a warehouse. |
| Which features use compute but not your warehouse? | Serverless features: Snowpipe, Automatic Clustering, materialized view maintenance, Search Optimization, serverless Tasks, QAS, replication. |
