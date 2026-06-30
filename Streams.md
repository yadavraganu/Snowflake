A Stream is a change data capture (CDC) object that tracks insert, update, and delete mutations made to a table, directory table, or view. It stores a transaction offset pointer allowing downstream pipelines to consume mutated rows exactly once.

## 1. Stream Types & Architecture

Snowflake streams do not store physical table data. Instead, they store a transactional offset tracking the differences between a fixed point in time and the current state of a source object.

| Stream Type | Syntax Option | Tracked Changes | Use Case |
| --- | --- | --- | --- |
| **Standard (Delta)** | *Default* | Inserts, Updates, Deletes. Shows net changes over time. | General purpose ELT/ETL pipeline staging tables. |
| **Append-Only** | `APPEND_ONLY = TRUE` | Inserts only. Ignores updates and deletes. | Immutable log processing, clickstream data, IoT ingestion. |
| **Insert-Only** | `INSERT_ONLY = TRUE` | Inserts only on overwrite operations (e.g., external tables). | Cloud storage file drop tracking via External Tables. |

### The "Net Effect" Evaluation Architecture

Standard streams evaluate the collective history of a row across the transaction window and only present the net delta.

```
[Row Inserted] ───> [Row Updated] ───> [Row Deleted] ───> Net Stream Output: (Empty)

```

* **Inserted + Updated:** Stream shows a single `INSERT` action containing the newest data state.
* **Inserted + Deleted (Before Consumption):** The row is completely skipped from the stream evaluation block ($0$ rows processed).
* **Multiple Updates:** Stream condenses these into one `DELETE` (original state) and one `INSERT` (latest state).

---

## 2. Stream Metadata Columns

When you query a stream, Snowflake appends three tracking metadata columns to the source object's schema:

| Metadata Column | Data Type | Value / Function |
| --- | --- | --- |
| **`METADATA$ACTION`** | `VARCHAR` | `INSERT` or `DELETE`. *(Note: An UPDATE is represented as a pair of rows: a DELETE row followed by an INSERT row).* |
| **`METADATA$ISUPDATE`** | `BOOLEAN` | `TRUE` if the action was part of an `UPDATE` statement; `FALSE` if it was a true standalone insert or delete. |
| **`METADATA$ROW_ID`** | `VARCHAR` | A unique, permanent hex key tracking the specific physical row across changes. |

## 3. Creating Streams (DDL)

### Method 1: Standard Stream on a Permanent Table

```sql
-- Track all modifications (Inserts, Updates, Deletes) on the orders table
CREATE OR REPLACE STREAM prod_db.raw.orders_stream 
  ON TABLE prod_db.raw.orders
  COMMENT = 'Captures transactional mutations on core orders table';

```

### Method 2: Append-Only Stream on a Staging Table

```sql
-- Optimize performance by ignoring updates/deletes on raw append-heavy logs
CREATE OR REPLACE STREAM prod_db.raw.logs_stream 
  ON TABLE prod_db.raw.web_logs
  APPEND_ONLY = TRUE;

```

### Method 3: Stream on an External Table (Insert-Only)

```sql
-- Tracks when new files are added to cloud object storage via an external table
CREATE OR REPLACE STREAM prod_db.raw.ext_storage_stream 
  ON EXTERNAL TABLE prod_db.raw.ext_s3_inventory
  INSERT_ONLY = TRUE;

```

### Method 4: Stream on a Secure View

```sql
-- Tracks data changes through an underlying view's logic 
CREATE OR REPLACE STREAM prod_db.analytics.v_orders_stream 
  ON VIEW prod_db.analytics.v_filtered_orders;

```

### Method 5: Append-Only Stream on a Materialized View

```sql
-- Point a stream at an asynchronously refreshed materialized view
CREATE OR REPLACE STREAM prod_db.analytics.mv_metrics_stream 
  ON MATERIALIZED VIEW prod_db.analytics.mv_daily_metrics
  APPEND_ONLY = TRUE;

```

## 4. Consuming Stream Data (DML & Transaction Locks)

Reading from a stream inside a Data Manipulation Language (DML) transaction advances its tracking offset. A standard `SELECT * FROM stream` statement **does not** advance the offset.

### The Single-Transaction Consumption Pattern

To clear records from a stream, you must process it within a transaction block (`INSERT INTO... SELECT`, `MERGE`, or `CREATE TABLE AS`).

```sql
-- Consumer Pattern: Merging a stream into a downstream target table
BEGIN TRANSACTION;

MERGE INTO prod_db.analytics.orders_dim target
USING prod_db.raw.orders_stream source
ON target.order_id = source.order_id
WHEN MATCHED AND source.METADATA$ACTION = 'INSERT' AND source.METADATA$ISUPDATE = TRUE THEN
  UPDATE SET target.amount = source.amount, target.updated_at = CURRENT_TIMESTAMP()
WHEN MATCHED AND source.METADATA$ACTION = 'DELETE' THEN
  DELETE
WHEN NOT MATCHED AND source.METADATA$ACTION = 'INSERT' THEN
  INSERT (order_id, customer_id, amount, created_at) 
  VALUES (source.order_id, source.customer_id, source.amount, source.created_at);

-- Once committed, the stream offset advances and these records disappear from the stream view
COMMIT;

```

### Multi-DML Consumption Split (Multi-Table Insert)

If you read a stream multiple times within **separate** DML transactions, each statement advances the stream independently. To populate multiple target tables using the *exact same data snapshot*, use a multi-table insert:

```sql
-- Multi-Table Insert: Consuming a single stream snapshot into two targets safely
INSERT ALL
  INTO prod_db.analytics.orders_bi SELECT order_id, amount WHERE METADATA$ACTION = 'INSERT'
  INTO prod_db.audit.orders_log SELECT order_id, METADATA$ACTION, CURRENT_TIMESTAMP()
SELECT * FROM prod_db.raw.orders_stream;

```
## 5. Streams + Tasks Orchestration (DAG Execution Patterns)

Combining Streams with Tasks allows you to implement event-driven ELT pipelines. By leveraging the `WHEN SYSTEM$STREAM_HAS_DATA()` evaluation clause, you guarantee that compute resources only spin up when there is actionable data to process.

```sql
-- Create an orchestrator Root Task running on a tight schedule
-- Snowflake evaluates the WHEN clause in the Cloud Services layer for $0 compute cost
CREATE OR REPLACE TASK prod_db.pipelines.process_users_root_task
  USER_TASK_MANAGED_INITIAL_WAREHOUSE_SIZE = 'SMALL' -- Serverless compute option
  SCHEDULE = '1 MINUTE' 
  WHEN SYSTEM$STREAM_HAS_DATA('prod_db.raw.orders_stream')
AS
  MERGE INTO prod_db.analytics.orders_dim target
  USING prod_db.raw.orders_stream source
  ON target.order_id = source.order_id
  WHEN MATCHED AND source.METADATA$ACTION = 'INSERT' AND source.METADATA$ISUPDATE = TRUE THEN
    UPDATE SET target.amount = source.amount, target.updated_at = CURRENT_TIMESTAMP()
  WHEN NOT MATCHED AND source.METADATA$ACTION = 'INSERT' THEN
    INSERT (order_id, amount, created_at) VALUES (source.order_id, source.amount, CURRENT_TIMESTAMP());

-- Activate the task
ALTER TASK prod_db.pipelines.process_users_root_task RESUME;

```
## 6. Advanced Controls & Time Travel Joins

### Initializing Streams at Specific Historical Points

You can create a stream that starts tracking changes from an exact historical point in time using Snowflake's Time Travel retention buffer.

```sql
-- Create a stream capturing changes starting precisely 2 hours ago
CREATE OR REPLACE STREAM orders_stream_historical 
  ON TABLE prod_db.raw.orders
  AT(OFFSET => -7200);

-- Create a stream starting right before a specific bad query executed
CREATE OR REPLACE STREAM orders_stream_rollback 
  ON TABLE prod_db.raw.orders
  BEFORE(STATEMENT => '01b45cd6-0000-1234-0000-000012345678');

```
## 7. Operational Monitoring & Verification

### Evaluating Stream Data Presence Natively

To avoid spinning up virtual warehouses just to scan empty tables, use the `SYSTEM$STREAM_HAS_DATA` function. This evaluates metadata directly in the Cloud Services layer for $0$ warehouse compute cost.

```sql
-- Returns TRUE if records are waiting to be consumed, FALSE if empty
SELECT SYSTEM$STREAM_HAS_DATA('prod_db.raw.orders_stream');

```

### Auditing Stream Lifecycles and Staleness

```sql
-- List all active streams, their type, target objects, and staleness states
SHOW STREAMS LIKE '%_stream';

-- Crucial output columns to check:
-- 'stale'       -> 'true' if the stream has gone past its retention window and cannot be consumed.
-- 'stale_after' -> The exact future timestamp when this stream will expire if not consumed.

```
## 8. Checking and Resetting Stream Offsets

Snowflake manages stream offset cursors internally via the Cloud Services metadata layer. While you cannot query a raw physical log sequence number, you can programmatically audit the offset's age, state, and target alignment, or force-advance the cursor when necessary.

### Tracking Offset Status via Information Schema

To monitor whether an offset is active or nearing expiration across an entire database catalog, query the Information Schema:

```sql
-- Track the exact expiration timestamps for all active stream offsets
SELECT 
    STREAM_CATALOG AS DATABASE_NAME,
    STREAM_SCHEMA AS SCHEMA_NAME,
    STREAM_NAME,
    TABLE_NAME AS SOURCE_OBJECT_NAME,
    STALE,         -- Returns TRUE if the offset has fallen past the Time Travel window
    STALE_AFTER    -- The exact timestamp when this offset cursor will permanently expire
FROM PROD_DB.INFORMATION_SCHEMA.STREAMS
WHERE STALE = 'FALSE';

```

### Calculating the Last Consumption Point

Because a stream offset locks the source table's micro-partitions up to a maximum of 14 days, you can calculate the exact historical timestamp of your last successful pipeline consumption using the `STALE_AFTER` metric:

$$\text{Last Advanced Timestamp} = \text{STALE\_AFTER} - \text{Table Retention Window (Max 14 Days)}$$

### The "Dummy Consume" Pattern (Resetting/Skipping an Offset)

If a downstream pipeline crashes permanently, or you need to deliberately skip a batch of bad data and reset the stream's offset cursor to the current time *without* modifying target production data, execute a false-predicate DML transaction:

```sql
-- Force-advance the stream offset cursor to 'NOW', clearing all pending rows
BEGIN TRANSACTION;
  
  -- Evaluates the stream metadata but writes 0 rows due to the '1 = 0' hard filter
  INSERT INTO prod_db.raw.orders (order_id, amount)
  SELECT order_id, amount 
  FROM prod_db.raw.orders_stream 
  WHERE 1 = 0; 

-- Committing this advances the transactional pointer, clearing the stream completely
COMMIT;

```
## 9. High-Scale MERGE Performance Tuning

Consuming a large volume of stream data via a `MERGE` statement can result in heavy micro-partition scanning if the execution plan isn't optimized.

* **Isolate and Filter Stream Mutation Subsets:** Group and deduplicate your stream rows using a Common Table Expression (CTE) or subquery so Snowflake evaluates exactly one row per unique ID.
* **Enforce Clustering on Join Keys:** Ensure both your target production table and your source streaming tables are explicitly clustered on the keys utilized in your `ON` match clause. This enables precise **partition pruning**.
* **Add a Value Change Guard Condition:** Prevent unnecessary micro-partition writes by adding a hash or equality check to ensure a value actually changed before applying an `UPDATE`.

```sql
-- Optimized Production Merge Layout
MERGE INTO analytics.orders_fact target
USING (
    -- Group mutations to present exactly one row per ID to the MERGE engine
    SELECT * FROM raw.orders_stream
    QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY created_at DESC) = 1
) source
ON target.order_id = source.order_id
-- Maximize pruning by verifying if a true modification occurred before hitting the disk
WHEN MATCHED AND source.METADATA$ACTION = 'INSERT' AND source.METADATA$ISUPDATE = TRUE 
  AND target.hash_check != source.hash_check THEN
    UPDATE SET target.payload = source.payload, target.hash_check = source.hash_check
WHEN NOT MATCHED AND source.METADATA$ACTION = 'INSERT' THEN
    INSERT (order_id, payload, hash_check) VALUES (source.order_id, source.payload, source.hash_check);

```
## 10. Governance, Security, & `ACCESS_HISTORY`

### Evaluating Stream Read Operations

When a data pipeline processes rows out of a stream, Snowflake logs this explicitly in the `SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY` system view, capturing both the stream object and the underlying source tables providing the micro-partitions.

```sql
-- Audit Query: Find out who is reading CDC data through streams
SELECT
    USER_NAME,
    ROLE_NAME,
    QUERY_ID,
    f.value:objectName::STRING AS ACCESSED_STREAM_NAME,
    f.value:objectDomain::STRING AS OBJECT_DOMAIN
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY,
LATERAL FLATTEN(input => DIRECT_OBJECTS_ACCESSED) f
WHERE f.value:objectDomain::STRING = 'Stream'
  AND QUERY_START_TIME >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY QUERY_START_TIME DESC;

```
### Masking Policy Inheritance

If a column in a source table has a **Column-Level Masking Policy** attached to it, that policy is fully inherited by the stream. Ensure your pipeline execution roles have explicit privileges to read unmasked data if downstream targets require plain text values.

## 11. Privileges & Access Control (RBAC)

```sql
-- 1. Allow a data engineer role to build streams inside a schema
GRANT CREATE STREAM ON SCHEMA prod_db.raw TO ROLE data_engineer;

-- 2. Required on the source table for a stream to read its changes
GRANT SELECT ON TABLE prod_db.raw.orders TO ROLE data_engineer;

-- 3. Required for downstream tasks or execution engines to select from the stream
GRANT SELECT ON STREAM prod_db.raw.orders_stream TO ROLE pipeline_execution_role;

```
## 12. Operational Nuances & Critical Restrictions

* **The Data Retention Staleness Trap:** A stream becomes **stale** if its transaction offset extends past the data retention period (`DATA_RETENTION_TIME_IN_DAYS`) of its source table. Snowflake automatically extends the table's effective retention period to protect streams from going stale, up to a maximum cap of **14 days**. If you do not consume data within that window, the stream goes permanently stale and cannot be recovered.
* **Materialized View Refresh Asynchrony:** Materialized Views are refreshed asynchronously by a background Snowflake process. There is a minor latency gap before data inserted into a base table populates the Materialized View, and a subsequent gap before it reflects in the View's stream. Pointing a stream at a Materialized View is strictly limited to **Append-Only** streams.
* **The Database Replication / Failover Gap:** When a database containing a stream is replicated to a secondary account for Disaster Recovery (DR), the stream's definition is copied, but its **transactional offset pointer is completely reset**. Upon region failover, streams will appear empty, missing any CDC data that accumulated on the primary region right before the failover event.
* **Schema Evolution (DDL Changes on Source Tables):** If columns tracked by a stream are dropped or structurally altered via DDL on the source table, the stream becomes structurally locked or returns a compilation error upon querying. Added columns are automatically appended to the stream output schema on subsequent mutations.

## 13. Critical Production Pitfalls

* **The Source Table Overwrite Destruction (`CREATE OR REPLACE`):** Executing a `CREATE OR REPLACE TABLE` on a source table drops that physical object and replaces it. This action **instantly breaks and invalidates any streams** attached to the original table. Use `TRUNCATE TABLE` or `INSERT OVERWRITE` instead to preserve stream continuity.
* **The Cross-Database Pipeline Block:** While you can create a stream in `Database_A` that targets a table in `Database_B`, you cannot replicate or share that stream via Snowflake Data Sharing across separate Snowflake accounts. Stream consumption is strictly local to the host account ecosystem.
* **The View Modification Failure:** If a stream is built on top of a View, any subsequent modifications to that view's underlying tables or column expressions will invalidate the stream tracking offset.
* **Empty Stream Virtual Warehouse Burn:** Avoid scheduling tasks to select from streams without a `WHEN SYSTEM$STREAM_HAS_DATA()` condition. Running a standard query scanning an empty stream still forces an active warehouse to run, resulting in unnecessary billing.
