Tasks allow you to execute a single SQL statement, a procedural Snowflake Scripting block, or a Stored Procedure on a defined cron/interval schedule or in response to a stream state change. Multiple tasks can be linked together into a Directed Acyclic Graph (DAG) to build complex data pipelines directly inside Snowflake.

## 1. Task Compute Models & Pricing

Snowflake tasks can run using either user-managed virtual warehouses or Snowflake-managed serverless compute resources.

| Compute Type | Configuration | Ingestion/Billing Model | Best For |
| --- | --- | --- | --- |
| **User-Managed** | `WAREHOUSE = 'my_wh'` | Standard warehouse per-second billing. | Heavy, predictable workloads; pipelines that can share an active warehouse. |
| **Serverless** | *Omit `WAREHOUSE` parameter* | Dynamically allocated by Snowflake. Billed on compute resource usage factor. | Light, bursty, or unpredictable jobs; tasks that run for short durations. |

## 2. Parameter Inheritance & Hierarchy

Tasks are schema-level objects. While they inherit default session parameters from the Schema, Database, and Account levels, operational limits like timeouts can be strictly tuned at the task level.

```
Account Level (Global defaults)
       └── Database Level
             └── Schema Level
                   └── Task Level (Lowest level overrides)

```

## 3. Creating Tasks (DDL)

### Method 1: Scheduled Task (Using User-Managed Warehouse)

```sql
-- Create a root task that runs every hour using standard cron syntax
CREATE OR REPLACE TASK prod_db.raw.parse_logs_task
  WAREHOUSE = prod_compute_wh
  SCHEDULE = 'USING CRON 0 * * * * UTC' -- Runs at minute 0 of every hour
  USER_TASK_TIMEOUT_MS = 1200000         -- Limits runtime to 20 minutes max
  SUSPEND_TASK_AFTER_NUM_FAILURES = 3   -- Auto-suspends root/DAG after 3 consecutive failures
  COMMENT = 'Parses raw stage logs into intermediate tables'
AS
  CALL prod_db.raw.sp_parse_json_logs();

```

### Method 2: Scheduled Task (Using Serverless Compute)

```sql
-- Omitting the WAREHOUSE parameter automatically targets serverless compute
CREATE OR REPLACE TASK prod_db.raw.cleanup_session_task
  SCHEDULE = '15 MINUTE' -- Simple numeric interval notation
  USER_TASK_MANAGED_INITIAL_WAREHOUSE_SIZE = 'SMALL' -- Advisory baseline sizing for first run
AS
  DELETE FROM prod_db.raw.session_cache WHERE expiry_time < CURRENT_TIMESTAMP();

```

### Method 3: Conditional Stream Dependency Task

```sql
-- This task fires on its schedule ONLY if change data capture data exists in the stream
CREATE OR REPLACE TASK prod_db.raw.process_stream_task
  WAREHOUSE = prod_compute_wh
  SCHEDULE = '5 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('prod_db.raw.orders_stream')
AS
  INSERT INTO prod_db.analytics.orders_fact 
  SELECT * FROM prod_db.raw.orders_stream WHERE METADATA$ACTION = 'INSERT';

```

## 4. Task Orchestration & DAG Pipelines (Dependency Linking)

To build a Directed Acyclic Graph (DAG), you define a **Root Task** (which contains the schedule) and link **Child Tasks** to it using the `AFTER` keyword. Child tasks cannot have their own `SCHEDULE`.

```
        [ Root Task: Run Schedule ] 
               /           \
              ▼             ▼
       [ Child Task A ]   [ Child Task B ]
              \             /
               ▼           ▼
        [ Child Task C (Joins A & B) ]
                    │
                    ▼
       [ Finalizer Task: FINALIZE = Root ] (Guaranteed Execution)

```

### Defining a Multi-Task DAG Pipeline with a Finalizer

A **Finalizer Task** is a special type of child task that is guaranteed to run at the very end of a DAG run, **regardless of whether the preceding tasks succeeded, failed, or timed out**.

```sql
-- 1. Create Root Task (Controls the trigger cadence)
CREATE OR REPLACE TASK pipeline_root_task
  WAREHOUSE = prod_compute_wh
  SCHEDULE = 'USING CRON 0 6 * * * America/New_York' -- Every day at 6:00 AM EST
AS
  CALL sp_extract_to_stage();

-- 2. Create First Dependent Child Task
CREATE OR REPLACE TASK pipeline_child_load
  WAREHOUSE = prod_compute_wh
  AFTER pipeline_root_task
AS
  COPY INTO raw_table FROM @my_s3_stage;

-- 3. Create Second Dependent Child Task (Runs only after Child 1 completes)
CREATE OR REPLACE TASK pipeline_child_transform
  WAREHOUSE = prod_compute_wh
  AFTER pipeline_child_load
AS
  INSERT INTO final_dimension SELECT * FROM raw_table;

-- 4. Create a Finalizer Task (Guaranteed to execute for environment cleanup or alerting)
CREATE OR REPLACE TASK pipeline_dag_cleanup
  WAREHOUSE = prod_compute_wh
  FINALIZE = pipeline_root_task -- Attaches exclusively to the root as a finalizer
AS
  CALL sp_drop_temporary_working_tables();

```

## 5. Lifecycle Control & Execution State

- **CRITICAL GUARDRAIL:** All tasks are created in a **`SUSPENDED`** state by default. A DAG will not execute until you manually resume its objects.

### Resuming and Suspending Individual Tasks

```sql
-- Activate a single standalone task
ALTER TASK parse_logs_task RESUME;

-- Pause a task
ALTER TASK parse_logs_task SUSPEND;

```

### Activating an Entire DAG Hierarchy

In a DAG, child tasks must be resumed **before** the root task. Resuming the root task first will cause child tasks to be skipped.

```sql
-- Recommended Method: Resume the root, all child tasks, and the finalizer at once
SELECT SYSTEM$TASK_DEPENDENTS_ENABLE('pipeline_root_task');

-- Manual fallback method (Bottom-Up):
ALTER TASK pipeline_dag_cleanup RESUME;      -- Resume Finalizer
ALTER TASK pipeline_child_transform RESUME; -- Resume Child 2
ALTER TASK pipeline_child_load RESUME;      -- Resume Child 1
ALTER TASK pipeline_root_task RESUME;       -- Resume Root last

```

## 6. Graph Flow Control, Retries, & Overlaps

### Task Graph Overlap Policies

You can control what happens when a DAG's scheduled execution time occurs while a previous run of that same DAG is still running. Set this parameter on the **Root Task**.

* `NO_OVERLAP` *(Default)*: The current running graph must completely finish before the next scheduled instance can start. If it's still running, the new scheduled run is skipped.
* `ALLOW_CHILD_OVERLAP`: The root task will not overlap, but child tasks from a new graph run can begin executing parallel instances alongside lagging child tasks from a prior run.
* `ALLOW_ALL_OVERLAP`: Snowflake spins up entirely concurrent instances of the complete DAG graph layout.

```sql
ALTER TASK pipeline_root_task SET OVERLAP_POLICY = ALLOW_CHILD_OVERLAP;

```

### Automatic Graph Retries

If a task graph fails midway, Snowflake can automatically retry the graph starting specifically from the failed task, avoiding re-running successful upstream work.

```sql
-- Configure the root task to automatically retry a failed graph run up to 3 times
ALTER TASK pipeline_root_task SET TASK_AUTO_RETRY_ATTEMPTS = 3;

```

### Manual Execution & Retry Forcing

```sql
-- Execute a task immediately, ignoring its defined schedule window
EXECUTE TASK pipeline_root_task;

-- Execute a runtime instance using a dynamic JSON configuration parameter overriding defaults
EXECUTE TASK pipeline_root_task USING CONFIG = $${"environment": "production", "batch_id": 4022}$$;

-- If a graph run failed or was canceled, manually force a retry from the exact failure point
EXECUTE TASK pipeline_root_task RETRY LAST;

```

## 7. Cloud Native Alerts & Error Handling

Instead of checking logs manually, you can configure tasks to automatically publish failure payloads directly to cloud notification infrastructure (AWS SNS, Azure Event Grid, GCP Pub/Sub).

```sql
-- Assign an error notification integration to push failure alerts natively
ALTER TASK pipeline_root_task SET ERROR_INTEGRATION = my_cloud_notification_int;

```

## 8. Operational Monitoring & Metadata Audits

### Querying Real-Time Graph Execution States

```sql
-- Check currently executing or scheduled graph runs across the account
SELECT * FROM TABLE(INFORMATION_SCHEMA.CURRENT_TASK_GRAPHS());

-- Retrieve details for graph runs completed in the last 60 minutes
SELECT * FROM TABLE(INFORMATION_SCHEMA.COMPLETE_TASK_GRAPHS(
    RESULT_LIMIT => 500,
    ROOT_TASK_NAME => 'PIPELINE_ROOT_TASK'
));

```

### Auditing Task Run Success & Error Messages

```sql
-- Evaluate success, failures, and actual pixel-perfect durations
SELECT 
    TASK_NAME,
    STATE, -- 'SUCCEEDED', 'FAILED', or 'SKIPPED'
    ERROR_CODE,
    ERROR_MESSAGE,
    SCHEDULED_TIME,
    QUERY_START_TIME,
    COMPLETED_TIME,
    DATEDIFF('seconds', QUERY_START_TIME, COMPLETED_TIME) AS RUNTIME_SECONDS
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    DATE_RANGE_START => DATEADD('hour', -24, CURRENT_TIMESTAMP())
))
ORDER BY SCHEDULED_TIME DESC;

```

### Tracking Long-Running Graph Hangs

```sql
-- Find tasks that are currently running and may be hung
SELECT * 
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY()) 
WHERE STATE = 'EXECUTING';

```

### Auditing Serverless Task Costs

```sql
-- Query Account Usage to isolate credit burn metrics for serverless automated orchestration tasks
SELECT 
    TASK_NAME,
    SUM(CREDITS_USED) AS TOTAL_SERVERLESS_CREDITS
FROM SNOWFLAKE.ACCOUNT_USAGE.SERVERLESS_TASK_HISTORY
WHERE START_TIME >= DATEADD('day', -30, CURRENT_DATE())
GROUP BY TASK_NAME
ORDER BY TOTAL_SERVERLESS_CREDITS DESC;

```

## 9. Privileges & Access Control (RBAC)

Task manipulation requires explicit management privileges because scheduled compute execution carries underlying cost liabilities.

```sql
-- 1. Allow a data engineer role to create tasks inside a schema
GRANT CREATE TASK ON SCHEMA prod_db.raw TO ROLE data_engineer;

-- 2. Required to execute or alter tasks (Allows RESUME / SUSPEND actions)
GRANT OPERATE ON TASK prod_db.raw.parse_logs_task TO ROLE data_engineer;

-- 3. Allow a role to view task configurations without editing rights
GRANT MONITOR ON TASK prod_db.raw.parse_logs_task TO ROLE data_analyst;

-- 4. Global Privilege: Required if using Serverless Task execution compute paths
GRANT EXECUTE TASK ON ACCOUNT TO ROLE data_engineer;

```

## 10. Constraints, Limits & Strict Restrictions

### Graph Architecture Limits

* **The Shared Ownership Constraint:** All tasks within a single Directed Acyclic Graph (DAG) **must share the exact same owner role** (i.e., one role must hold the `OWNERSHIP` privilege over every task in the graph) and must reside within the same database and schema.
* **Graph Modification Lockout:** You cannot add a child task (`AFTER`), remove a predecessor link, change a schedule, or modify an existing task's structural parameters if the **Root Task** of that graph is currently active (`STARTED`). You must `SUSPEND` the root task first.
* **Maximum Graph Depth/Size:** A single DAG is limited to a maximum of **1,000 tasks total** (including the root task). No single task within the graph can have more than **100 predecessor tasks** or **100 child tasks**.

### Execution & Runtime Restrictions

* **Single SQL Statement Limit:** A single task can only execute **exactly one** SQL statement or call a single stored procedure/Snowflake Scripting anonymous block. If you need to run multiple sequential commands, you must wrap them inside a Stored Procedure or separate them into multiple dependent child tasks.
* **Hard Timeout Max Cap:** The absolute maximum value for the `USER_TASK_TIMEOUT_MS` parameter is **3,600,000 milliseconds (1 hour)**. If a task exceeds this limit, Snowflake forcibly aborts the execution regardless of the parameter configuration.
* **Cron Interval Floor:** While simple numeric schedules can be set down to `1 MINUTE`, standard cron schedules have a minimum execution interval floor of **1 minute**.

### Serverless vs. User-Managed Restrictions

* **Serverless Privilege Cap:** To deploy or execute a serverless task, the executing role *must* possess the global account privilege `EXECUTE TASK ON ACCOUNT`. Standard tasks utilizing user-managed warehouses only require schema-level `CREATE TASK` and warehouse `USAGE` privileges.
* **Account-Level Cost Safety:** Serverless compute resources do not adhere to standard warehouse resource monitors. Instead, serverless tasks must be managed via specific parameters or account configurations to prevent unexpected credit consumption.

### Stream & Condition Evaluation Constraints

* **The Stream Evaluation Tax:** When using `WHEN SYSTEM$STREAM_HAS_DATA('stream_name')`, Snowflake utilizes the Cloud Services layer to evaluate the stream's metadata on your specified schedule. If a task is scheduled to run every 1 minute, Snowflake evaluates that stream 1,440 times a day. If the stream is consistently empty, this evaluation frequency will still trigger minor cloud services billing allocations.
* **No Multi-Stream Complex Logic:** The `WHEN` clause can evaluate basic boolean logic, but it cannot contain subqueries, UDFs, or complex multi-table joins. It is strictly optimized for fast, metadata-only lookups.

## 11. Critical Production Pitfalls

* **The Object Overwrite Trap (`CREATE OR REPLACE`):** Executing a `CREATE OR REPLACE TASK` statement drops the underlying task metadata definition and builds a fresh object. The task will revert back to a **`SUSPENDED`** state, instantly halting any automated pipeline schedules until it is explicitly resumed.
* **The Dropped Predecessor Broken Chain:** If an intermediary child task inside a complex DAG is dropped or replaced, the chain breaks. The downstream child tasks linked to it will become un-orphaned or fail to trigger because their immediate predecessor reference no longer exists.
* **Dangling Tasks Billed on Streams:** When a task relies on a `WHEN SYSTEM$STREAM_HAS_DATA()` clause, ensure your schedule interval matches your true data latency requirement to minimize unnecessary check frequencies.
