## 1. Overview: Three Tools, One Workflow

| Tool | Question It Answers | Granularity | Where |
| --- | --- | --- | --- |
| **Query History** | *What ran, who ran it, when, on which warehouse, how long, did it fail?* | One row per query | Snowsight, `INFORMATION_SCHEMA`, `ACCOUNT_USAGE` |
| **Query Profile** | *How did Snowflake actually execute this query, and where did the time go?* | Operator-level execution graph of one query | Snowsight (Query Profile tab), `GET_QUERY_OPERATOR_STATS` |
| **Query Insights** | *Did Snowflake detect a known condition that hurts performance, and what should I try?* | Condition-level hints for one query | Snowsight (Query Insights pane), `QUERY_INSIGHTS` view |

```
   DETECT                    DIAGNOSE                         FIX              VERIFY
 ┌──────────────┐      ┌─────────────────────┐        ┌────────────┐    ┌──────────────┐
 │ Query History│ ───► │ Query Profile       │ ─────► │ Rewrite SQL│───►│ Re-run, then │
 │ (find slow / │      │ + Query Insights    │        │ cluster /  │    │ compare in   │
 │ costly / bad │      │ (why is it slow?)   │        │ resize /   │    │ Query History│
 │ queries)     │      └─────────────────────┘        │ tune WH    │    └──────────────┘
 └──────────────┘                                     └────────────┘
```

**Rule of thumb:** use **History** to *find* the problem query, **Profile** to *understand* it, **Insights** as a *quick hint*, and **History again** to *prove* the fix worked.

<br></br>

## 2. Query History

### 2.1 Where to Find It

| Source | Retention | Latency | Scope | Best For |
| --- | --- | --- | --- | --- |
| **Snowsight** (Monitoring > Query History) | Roughly the last **14 days** | Near real time | Your queries by default; more with the right privileges | Interactive investigation, opening Query Profile |
| **`INFORMATION_SCHEMA.QUERY_HISTORY*`** table functions | Last **7 days** | Near real time | Current account; returns your queries unless your role has wider privileges | Live monitoring, scripts, recently completed or running queries |
| **`SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY`** view | Up to **365 days** | Up to about **45 minutes** | Whole account | Trend analysis, cost and performance reporting, auditing |

Table functions in `INFORMATION_SCHEMA`:

| Function | Scope |
| --- | --- |
| `QUERY_HISTORY` | Account (subject to privileges) |
| `QUERY_HISTORY_BY_SESSION` | A single session |
| `QUERY_HISTORY_BY_USER` | A single user |
| `QUERY_HISTORY_BY_WAREHOUSE` | A single warehouse |

```sql
-- Near real time: last hour, up to 100 queries
SELECT query_id, user_name, warehouse_name, execution_status, total_elapsed_time
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
       END_TIME_RANGE_START => DATEADD('hour', -1, CURRENT_TIMESTAMP()),
       RESULT_LIMIT         => 100))
ORDER BY start_time DESC;

-- One warehouse only
SELECT *
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_WAREHOUSE(
       WAREHOUSE_NAME => 'BI_WH', RESULT_LIMIT => 50));
```

**Access:** `ACCOUNT_USAGE` requires the `SNOWFLAKE` database privileges (e.g., `IMPORTED PRIVILEGES` on the `SNOWFLAKE` database or the appropriate database role such as `USAGE_VIEWER`). Seeing other users' queries in Snowsight or the table functions also depends on your role (for example, `ACCOUNTADMIN`, or a role granted `MONITOR` on the warehouse).

### 2.2 Snowsight Query History Features

* **Filters:** user, warehouse, status, duration, SQL text, query ID, query tag, query type, time range.
* **Query Details:** status, user, role, warehouse, start/end, and the **duration breakdown** (queued, compilation, execution).
* **SQL Text** tab and **Query Profile** tab per query.
* **Grouping** of related and repeated queries to spot recurring heavy statements.

### 2.3 Important `QUERY_HISTORY` Columns

| Category | Columns | Notes |
| --- | --- | --- |
| **Identity** | `QUERY_ID`, `QUERY_TEXT`, `QUERY_TYPE`, `QUERY_TAG`, `SESSION_ID` | `QUERY_TEXT` is truncated for very large statements (about 100K characters) |
| **Who / where** | `USER_NAME`, `ROLE_NAME`, `WAREHOUSE_NAME`, `WAREHOUSE_SIZE`, `WAREHOUSE_TYPE`, `CLUSTER_NUMBER` | `CLUSTER_NUMBER` shows which cluster of a multi-cluster warehouse ran it |
| **Status** | `EXECUTION_STATUS`, `ERROR_CODE`, `ERROR_MESSAGE` | Typical values: `SUCCESS`, `FAIL`, `INCIDENT` |
| **Timing (ms)** | `START_TIME`, `END_TIME`, `TOTAL_ELAPSED_TIME`, `COMPILATION_TIME`, `EXECUTION_TIME` | Total includes queue, compile, and execute phases |
| **Queuing (ms)** | `QUEUED_PROVISIONING_TIME`, `QUEUED_REPAIR_TIME`, `QUEUED_OVERLOAD_TIME`, `TRANSACTION_BLOCKED_TIME` | Provisioning = waiting for warehouse start; Overload = warehouse too busy; Blocked = waiting on a transaction lock |
| **Data volume** | `BYTES_SCANNED`, `ROWS_PRODUCED`, `BYTES_WRITTEN`, `ROWS_INSERTED / UPDATED / DELETED` | |
| **Pruning** | `PARTITIONS_SCANNED`, `PARTITIONS_TOTAL` | Ratio close to 1 on a selective query signals poor pruning |
| **Caching** | `PERCENTAGE_SCANNED_FROM_CACHE` | Fraction of scanned data served from the warehouse local cache |
| **Memory pressure** | `BYTES_SPILLED_TO_LOCAL_STORAGE`, `BYTES_SPILLED_TO_REMOTE_STORAGE` | Remote spill is the costly one |
| **Network** | `BYTES_SENT_OVER_THE_NETWORK` | |
| **Cost** | `CREDITS_USED_CLOUD_SERVICES` | Warehouse credits are **not** per query here (see section 2.6) |
| **Grouping** | `QUERY_HASH`, `QUERY_PARAMETERIZED_HASH` | Hash of query text / of text with literals normalized, for finding repeated patterns |
| **Acceleration** | `QUERY_ACCELERATION_BYTES_SCANNED` | Work offloaded by Query Acceleration Service |

### 2.4 Anatomy of Query Time

```
 TOTAL_ELAPSED_TIME
 ├── QUEUED  (provisioning + repair + overload)   <- waiting for compute
 ├── TRANSACTION_BLOCKED_TIME                      <- waiting on locks
 ├── COMPILATION_TIME                              <- Cloud Services: parse, optimize, plan
 └── EXECUTION_TIME                                <- warehouse doing the work
```

| Dominant Component | What It Usually Means | First Actions |
| --- | --- | --- |
| **Queued overload** | Warehouse out of capacity | Multi-cluster, split workloads, resize |
| **Queued provisioning** | Warehouse was suspended / resuming or resizing | Tune auto-suspend, keep warm for latency-sensitive work |
| **Transaction blocked** | Lock contention (e.g., concurrent `UPDATE` / `MERGE` on same table) | Serialize or batch writes, shorten transactions |
| **Compilation** | Very complex SQL, many nested views, huge column lists | Simplify, materialize intermediates, avoid `SELECT *` on wide views |
| **Execution** | Actual data processing | Open **Query Profile** |

### 2.5 Useful History Queries

```sql
-- 1. Slowest successful queries, last 24 hours
SELECT query_id, user_name, warehouse_name, warehouse_size,
       ROUND(total_elapsed_time/1000, 1)    AS elapsed_s,
       ROUND(queued_overload_time/1000, 1)  AS queued_s,
       ROUND(compilation_time/1000, 1)      AS compile_s,
       ROUND(execution_time/1000, 1)        AS exec_s,
       ROUND(bytes_scanned/POW(1024,3), 2)  AS gb_scanned,
       partitions_scanned, partitions_total
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
  AND execution_status = 'SUCCESS'
ORDER BY total_elapsed_time DESC
LIMIT 20;

-- 2. Where time goes, by warehouse (averages, last 7 days)
SELECT warehouse_name,
       COUNT(*)                              AS queries,
       ROUND(AVG(queued_overload_time)/1000, 2) AS avg_queued_s,
       ROUND(AVG(compilation_time)/1000, 2)     AS avg_compile_s,
       ROUND(AVG(execution_time)/1000, 2)       AS avg_exec_s
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND warehouse_name IS NOT NULL
GROUP BY warehouse_name
ORDER BY avg_queued_s DESC;

-- 3. Queries that spilled to remote storage
SELECT query_id, warehouse_name, warehouse_size,
       ROUND(bytes_spilled_to_local_storage/POW(1024,3), 2)  AS gb_spill_local,
       ROUND(bytes_spilled_to_remote_storage/POW(1024,3), 2) AS gb_spill_remote,
       ROUND(total_elapsed_time/1000, 1) AS elapsed_s
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND bytes_spilled_to_remote_storage > 0
ORDER BY bytes_spilled_to_remote_storage DESC
LIMIT 50;

-- 4. Repeated query patterns that consume the most time (literals normalized)
SELECT query_parameterized_hash,
       COUNT(*)                            AS runs,
       ROUND(SUM(total_elapsed_time)/1000) AS total_s,
       ROUND(AVG(total_elapsed_time)/1000, 1) AS avg_s,
       MIN(LEFT(query_text, 150))          AS sample_text
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND query_type = 'SELECT'
GROUP BY query_parameterized_hash
ORDER BY total_s DESC
LIMIT 25;

-- 5. Failed queries and why
SELECT error_code, LEFT(error_message, 120) AS error_message, COUNT(*) AS failures
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND execution_status = 'FAIL'
GROUP BY error_code, LEFT(error_message, 120)
ORDER BY failures DESC;

-- 6. Cache effectiveness by warehouse
SELECT warehouse_name,
       ROUND(AVG(percentage_scanned_from_cache) * 100, 1) AS avg_pct_from_cache
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND bytes_scanned > 0
GROUP BY warehouse_name;

-- 7. Poor pruning on large tables
SELECT query_id, partitions_scanned, partitions_total,
       ROUND(partitions_scanned / NULLIF(partitions_total, 0), 3) AS scan_ratio,
       LEFT(query_text, 120) AS query_start
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND partitions_total > 1000
  AND partitions_scanned / NULLIF(partitions_total, 0) > 0.8
ORDER BY bytes_scanned DESC
LIMIT 50;
```

### 2.6 Per-Query Cost Attribution

`QUERY_HISTORY` does not give credits per query for warehouse compute. Use:

| View | Purpose |
| --- | --- |
| `ACCOUNT_USAGE.QUERY_ATTRIBUTION_HISTORY` | **Compute credits attributed to each query** (`CREDITS_ATTRIBUTED_COMPUTE`), plus query acceleration credits. Excludes **idle** warehouse time. Higher latency than `QUERY_HISTORY` (hours). Very short queries may not appear |
| `ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY` | Total credits per warehouse per hour (includes idle time) |

```sql
-- Credits by query tag / user (last 30 days)
SELECT COALESCE(NULLIF(query_tag, ''), '(untagged)') AS query_tag,
       user_name,
       ROUND(SUM(credits_attributed_compute), 2) AS credits
FROM snowflake.account_usage.query_attribution_history
WHERE start_time >= DATEADD(day, -30, CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY credits DESC
LIMIT 50;
```

### 2.7 Query IDs, Tags, and Result Reuse

```sql
SELECT LAST_QUERY_ID();                          -- ID of your previous statement
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));  -- reuse stored result (about 24 hours)

ALTER SESSION SET QUERY_TAG = 'team=finance;job=nightly_load';  -- label queries for attribution
```

* **Query ID:** unique UUID for each execution.
* **Query tag:** free-text label per session/statement. Use a consistent convention (key=value pairs) so cost and performance can be grouped by team, job, or dashboard.
* **`QUERY_HASH` vs `QUERY_PARAMETERIZED_HASH`:** the first groups identical text, the second groups queries that differ only in literal values.

<br></br>

## 3. Query Profile

### 3.1 What It Is

A graphical, post-execution view of a **single query's execution plan with actual statistics**. It shows the **operators** (nodes) Snowflake used, how data flowed between them, and where time and resources were spent.

**How to open it:** Snowsight > Monitoring > Query History > select the query > **Query Profile** tab (also available while a long query is still running, with partial statistics). The profile is only kept for a limited period (about 14 days in Snowsight).

### 3.2 Layout

```
 ┌───────────────────────────────────────────────────────────────────┐
 │  QUERY PROFILE                                                      │
 │                                                                     │
 │   [Result]  (final output rows)                                     │
 │      ▲                                                              │
 │   [Aggregate]  GROUP BY customer_id             12%                 │
 │      ▲  2.1M rows                                                   │
 │   [Join]  INNER  o.cust_id = c.id               41%   <- hot node  │
 │      ▲ 980M rows              ▲ 5M rows                             │
 │  [TableScan] orders   35%   [TableScan] customers  3%               │
 │   partitions 9,100/9,500    partitions 40/40                        │
 │                                                                     │
 │  Right pane: Profile Overview | Statistics | Most Expensive Nodes   │
 └───────────────────────────────────────────────────────────────────┘
```

* **Nodes** = operators, showing the **percentage of total execution time**.
* **Arrows (edges)** = data flowing between operators, labelled with **row counts**.
* **Right-hand panel:** overall statistics, time breakdown, and the list of **Most Expensive Nodes**.
* Clicking a node shows its **attributes and statistics**.

### 3.3 Profile Overview: Execution Time Categories

| Category | Meaning | High Value Suggests |
| --- | --- | --- |
| **Processing** | CPU time spent on data processing | Good: work is compute-bound. Heavy compute may need a bigger warehouse or better SQL |
| **Local Disk IO** | Waiting on the warehouse SSD (cache reads/writes, spill to local disk) | Memory pressure or large scans reading cached data |
| **Remote Disk IO** | Waiting on remote storage (cold reads, remote spill) | Cold cache or **remote spilling**; low cache reuse |
| **Network Communication** | Waiting on data exchange between nodes (shuffles) | Large joins/aggregations redistributing data |
| **Synchronization** | Coordination between parallel workers | Skew or uneven work among nodes |
| **Initialization** | Query setup before processing starts | Usually small; large values point to compilation or setup overhead |

### 3.4 Profile Overview: Key Statistics

| Statistic | What to Look At |
| --- | --- |
| **Bytes scanned** | Total data read |
| **Percentage scanned from cache** | Warm-cache effectiveness |
| **Partitions scanned / Partitions total** | **Pruning quality**. Scanned close to total on a selective query means poor pruning |
| **Bytes spilled to local / remote storage** | Memory pressure. Remote spill is a strong "scale up or rewrite" signal |
| **Bytes sent over the network** | Data shuffling volume |
| **Bytes written / rows inserted, updated, deleted** | DML footprint |
| **Number of rows** (on edges) | Row explosion or effective reduction between steps |

### 3.5 Common Operators

| Operator | What It Does | What to Check |
| --- | --- | --- |
| **TableScan** | Reads micro-partitions from a table | Partitions scanned vs total, bytes scanned, % from cache, columns accessed, pushed-down filters |
| **Filter** | Applies a `WHERE`/`HAVING` predicate | Rows in vs out; if almost nothing is removed, the filter is not selective |
| **Join** (inner / outer / semi / anti) | Combines two inputs | Join condition, **output rows vs input rows** (explosion), build vs probe sizes |
| **CartesianJoin** | Join with no usable equality condition | Almost always a bug or a missing predicate; row count blows up |
| **JoinFilter** | Runtime filter that prunes one join side using the other | Good sign: dynamic pruning is working |
| **Aggregate / GroupingSets** | `GROUP BY`, `DISTINCT`, aggregates | High-cardinality grouping, memory use, spilling |
| **Sort / SortWithLimit** | `ORDER BY` / top-N (`ORDER BY ... LIMIT`) | Sorting huge inputs without a limit; spill |
| **WindowFunction** | Analytic functions (`OVER`) | Large partitions, big sorts |
| **UnionAll** | Combines inputs without dedup | `UNION` (without `ALL`) adds a dedup aggregate; prefer `UNION ALL` when duplicates are fine |
| **Flatten** | Expands `VARIANT` arrays / objects | Row multiplication from large arrays |
| **Generator** | Produces rows (e.g., `GENERATOR`, sequences) | Intended row count |
| **WithClause / WithReference** | CTE definition and references | A CTE referenced multiple times may be recomputed or materialized; check cost |
| **Insert / Update / Delete / Merge / CopyInto** | DML and loading | Partitions rewritten, rows affected |
| **Result** | Final result delivery | Rows returned and result size |

Special markers: a profile that shows **Query Result Reuse** means the result cache served the query (no warehouse work), and **metadata-based** results indicate the answer came from metadata.

### 3.6 Reading a Profile: Step-by-Step

1. **Check the overview first.** Which time category dominates (Processing, Local IO, Remote IO, Network)? Is anything spilling?
2. **Open "Most Expensive Nodes"** to find the hot operators.
3. **Inspect every TableScan:** compare partitions scanned vs total and bytes scanned. Poor pruning here is the most common and most fixable problem.
4. **Follow row counts along edges.** Where do row counts explode (joins, flatten) or fail to shrink (filters applied too late)?
5. **Inspect Joins:** are the conditions correct? Is the larger table on the right side? Is there a CartesianJoin?
6. **Look for spilling** in Aggregate, Sort, Join, and WindowFunction nodes.
7. **Form a hypothesis, change one thing, re-run, and compare** in Query History.

### 3.7 Common Problem Patterns

| Pattern in Profile | Likely Cause | Typical Fix |
| --- | --- | --- |
| TableScan: partitions scanned about equal to total on a selective filter | Filter on unclustered column or non-sargable predicate | Rewrite filter (no functions on column), cluster on filter column, use Search Optimization for point lookups |
| Join output rows far larger than inputs ("exploding join") | Missing or wrong join condition, duplicate keys on both sides | Fix join keys, dedupe before join, add predicates |
| CartesianJoin node | No equality predicate between inputs | Add join condition, check for accidental cross joins |
| Large remote spill | Working set larger than warehouse memory | Scale up warehouse, reduce columns/rows earlier, split query, Snowpark-optimized for extreme cases |
| High Remote Disk IO, low "% scanned from cache" | Cold cache or too much data per query | Keep warehouse warm, dedicate warehouse to repeated workload, reduce scan volume |
| High Network Communication | Large shuffles from big joins/aggregations | Filter early, reduce join width (select needed columns), pre-aggregate |
| Filter near the root removing most rows | Predicate not pushed down to scan | Rewrite so the filter applies on base tables; check views/CTEs |
| `UNION` followed by a heavy Aggregate | Implicit dedup | Use `UNION ALL` if duplicates are acceptable |
| Sort of a huge input | Unnecessary or unbounded `ORDER BY` | Remove, or add `LIMIT`, or sort only the final small result |
| Flatten produces massive row counts | Large nested arrays | Filter before flatten, extract needed paths only |
| One node consuming most of the time, small data | Inefficient expression, UDF, or external function | Simplify expression, replace UDF with native SQL |

### 3.8 Programmatic Access to Operator Statistics

```sql
-- Operator-level stats for the last query in your session
SELECT *
FROM TABLE(GET_QUERY_OPERATOR_STATS(LAST_QUERY_ID()));

-- Or for a specific query
SELECT operator_id, operator_type, operator_statistics, execution_time_breakdown
FROM TABLE(GET_QUERY_OPERATOR_STATS('<query_id>'))
ORDER BY step_id, operator_id;
```

Returns one row per operator, with `OPERATOR_TYPE`, `OPERATOR_STATISTICS` (rows, bytes, pruning stats, spilling), `EXECUTION_TIME_BREAKDOWN`, and `OPERATOR_ATTRIBUTES`. Useful for automating analysis or storing profiles beyond the Snowsight retention window.

### 3.9 `EXPLAIN` vs Query Profile

| Aspect | `EXPLAIN` | Query Profile |
| --- | --- | --- |
| Executes the query? | **No** | Query already ran (or is running) |
| Statistics | Estimated plan info (e.g., partitions assigned vs total, bytes assigned) | **Actual** rows, bytes, times, spill |
| Use for | Checking pruning and plan shape **before** an expensive run | Finding the real bottleneck **after** a run |

```sql
EXPLAIN USING TEXT SELECT ...;   -- also USING TABULAR / JSON
```

<br></br>

## 4. Query Insights

### 4.1 What It Is

**Query Insights** automatically detects **known conditions** that can affect a query's performance and returns:

* a **message** describing the condition and how it may affect performance,
* **details** about the part of the query that caused it, and
* a **suggested next step**, when the condition is harmful.

### 4.2 Where to Find It

| Location | How |
| --- | --- |
| **Snowsight** | Query History > select query > **Query Profile** tab > **Query Insights** pane. Select **View** beside an insight for details and **Learn more** for documentation |
| **SQL** | `SNOWFLAKE.ACCOUNT_USAGE.QUERY_INSIGHTS` view |

```sql
-- Explore what is available (confirm column names in your account)
DESC VIEW snowflake.account_usage.query_insights;
SELECT * FROM snowflake.account_usage.query_insights LIMIT 20;
```

Like other `ACCOUNT_USAGE` views, it is subject to latency and requires access to the `SNOWFLAKE` database.

### 4.3 Insight Types

| Insight | Meaning | Typical Action |
| --- | --- | --- |
| **No filter on table scan** (`QUERY_INSIGHT_NO_FILTER_ON_TOP_OF_TABLE_SCAN`) | A query or subquery has no `WHERE` clause, so it scans the whole table and may return more rows than intended | Add a `WHERE` clause to reduce data scanned |
| **Filter not applicable** (`QUERY_INSIGHT_INAPPLICABLE_FILTER_ON_TABLE_SCAN`) | A `WHERE` clause removes no rows, so more data may be scanned than intended | Review the predicate; make it effective or remove it |
| **Filter not selective** (`QUERY_INSIGHT_UNSELECTIVE_FILTER`) | The filter keeps most rows | Add more selective predicates, or accept that a large scan is needed |
| **LIKE filter with leading wildcard** | `LIKE '%text'` prevents effective pruning | Use prefix patterns, restructure the search, or consider Search Optimization |
| **Filter uses clustering key** (`QUERY_INSIGHT_FILTER_WITH_CLUSTERING_KEY`) | Informational: the filter aligns with the table's clustering key | Confirms clustering is being used |
| **Query benefited from search optimization** (and Snowflake Optima) | Informational: an optimization accelerated the query | Confirms the feature is paying off |

*The set of insight types has been growing. Check Snowflake's "Using query insights to improve performance" page for the current list.*

### 4.4 How to Use Insights Well

* Treat them as **hints, not verdicts**. A full-table scan with no filter is fine for a legitimate full aggregation.
* Use them to **triage quickly**, then confirm in the **Query Profile** (partitions scanned, row counts).
* **Positive insights** (clustering key used, search optimization used) are evidence that investments are working, and help justify their cost.
* Insights complement, not replace, other advisors: `QUERY_ACCELERATION_ELIGIBLE` (QAS candidates), warehouse load history (queuing), and clustering metrics.

<br></br>

## 5. Putting It Together: Diagnosis Playbook

| Symptom | Start In | Look For | Likely Fix |
| --- | --- | --- | --- |
| Dashboard suddenly slow | **History** (filter by warehouse and time) | Queuing, new heavy queries, changed warehouse size | Add clusters, isolate workloads |
| One query slow | **Profile** | Hot node, partitions scanned, spill, exploding join | Rewrite SQL, cluster, resize |
| Query slow only the first time | **History** (`PERCENTAGE_SCANNED_FROM_CACHE`, provisioning time) | Cold cache, warehouse resume | Tune auto-suspend, warm warehouse |
| Query waits before starting | **History** (queued columns) | Overload / provisioning / blocked | Multi-cluster, size, lock contention fixes |
| Slow compile, fast execute | **History** (`COMPILATION_TIME`) | Complex view stacks, giant SQL | Simplify, materialize intermediates |
| Bill spike | **History + Attribution** | Top queries by credits, untagged queries, new jobs | Right-size, fix top offenders, tag everything |
| Frequent identical queries | **History** (parameterized hash) | High run count, same text | Result caching, materialized views, tune once |
| Query fails | **History** (error code and message) | Timeouts, resource limits, permission errors | Fix per error, adjust `STATEMENT_TIMEOUT_IN_SECONDS` |
| "Why didn't clustering help?" | **Profile + Insights** | Partitions scanned vs total, filter insight | Fix non-sargable filter, review key |

<br></br>

## 6. Benchmarking and Testing Tips

To compare "before" and "after" fairly, control for caching:

```sql
-- Prevent result cache from answering repeat runs
ALTER SESSION SET USE_CACHED_RESULT = FALSE;

-- Test cold-cache performance: suspend then resume the warehouse
ALTER WAREHOUSE test_wh SUSPEND;
ALTER WAREHOUSE test_wh RESUME;
```

* Run each variant **several times**; report cold and warm timings separately.
* Use the **same warehouse size** for both versions.
* Tag test queries (`QUERY_TAG`) to find them easily in History.
* Compare **partitions scanned, bytes scanned, spill, and elapsed time**, not only elapsed time.
* Reset `USE_CACHED_RESULT` to `TRUE` afterwards.

<br></br>

## 7. Best Practices and Pitfalls

**Do**

* Use a consistent **query tag** convention in every job, BI connection, and service account.
* Use **`ACCOUNT_USAGE`** for trends and **`INFORMATION_SCHEMA`** for live monitoring.
* Review the **top N queries by time and by credits** weekly.
* Save important profiles (via `GET_QUERY_OPERATOR_STATS`) before they age out.
* Always check **pruning, spilling, and row counts** before resizing a warehouse.

**Don't**

* Don't assume a slow query needs a bigger warehouse; check the profile first.
* Don't judge by elapsed time alone; queued time may be the real culprit.
* Don't rely on `QUERY_HISTORY` to show per-query warehouse credits; use attribution views.
* Don't expect `ACCOUNT_USAGE` data instantly; allow for latency.
* Don't benchmark with the result cache on, or you will measure the cache, not your change.
* Don't treat Query Insights as definitive: verify in the profile.

<br></br>

## 8. Quick Interview Q&A

| Question | Short Answer |
| --- | --- |
| What is the Query Profile? | A graphical, post-execution view of a query's operator tree with actual statistics (time share, row counts, pruning, spilling). |
| How do you find a slow query's bottleneck? | Open the Query Profile, check the overview time categories, the most expensive nodes, TableScan pruning, spilling, and row counts on join edges. |
| What does "partitions scanned vs total" tell you? | How well pruning worked. Close to total on a selective query indicates poor pruning. |
| What is an exploding join? | A join whose output has far more rows than its inputs, usually from a missing or incorrect condition or duplicate keys. |
| Which spill is worse, local or remote? | Remote, because it goes to remote storage and is much slower. |
| Where can you see query history? | Snowsight, `INFORMATION_SCHEMA.QUERY_HISTORY*` functions (7 days, near real time), and `ACCOUNT_USAGE.QUERY_HISTORY` (365 days, up to about 45 min latency). |
| How are `QUERY_HISTORY` timing columns related? | Total elapsed = queued + blocked + compilation + execution. |
| What do the queued columns mean? | Provisioning: warehouse starting. Repair: warehouse repair. Overload: warehouse too busy. Plus transaction-blocked time for lock waits. |
| How do you get credits per query? | `QUERY_ATTRIBUTION_HISTORY` (compute credits attributed to each query, excluding idle time). |
| What is Query Insights? | An automatic detector of known performance-affecting conditions, with a message and suggested next step, shown in the Query Profile tab and the `QUERY_INSIGHTS` view. |
| Give examples of insights. | No filter on table scan, filter not applicable, filter not selective, LIKE with leading wildcard; also positive ones like "filter uses clustering key". |
| EXPLAIN vs Query Profile? | EXPLAIN shows the plan without running; Profile shows actual execution statistics after running. |
| How to group repeated queries? | `QUERY_HASH` for identical text, `QUERY_PARAMETERIZED_HASH` ignoring literal values. |
| How do you benchmark correctly? | Disable `USE_CACHED_RESULT`, control warehouse cache (suspend/resume for cold tests), same size, multiple runs, compare pruning and spill as well as time. |
| What does a Query Result Reuse node mean? | The result cache served the query; no warehouse compute was used. |
| What does high "Remote Disk IO" suggest? | Cold cache or remote spilling. |
| How do you retrieve operator stats in SQL? | `GET_QUERY_OPERATOR_STATS(query_id)`. |
