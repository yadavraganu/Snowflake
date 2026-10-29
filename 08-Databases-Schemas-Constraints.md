# Snowflake Databases, Schemas, Types & Constraints
<br></br>
## 1. The Object Hierarchy and Namespaces

```
 ORGANIZATION
   └── ACCOUNT  (one cloud, one region)
         ├── Account-level objects: users, roles, warehouses, integrations, network policies, resource monitors, shares
         └── DATABASE
               ├── Database roles
               └── SCHEMA
                     └── Schema objects: tables, views, stages, file formats, sequences,
                         streams, tasks, pipes, functions, procedures, policies, tags ...
```

| Level | Purpose |
| --- | --- |
| **Database** | Top-level logical container of schemas; the unit for cloning, replication, failover, and sharing |
| **Schema** | Logical grouping of objects inside a database; the usual unit for access control and organisation |
| **Object** | Table, view, stage, etc. Always lives in exactly one schema |

**Fully qualified name:** `database.schema.object`.  
A session has a current database and schema, which resolve partially qualified names.

```sql
USE DATABASE sales;            -- sets current database
USE SCHEMA sales.curated;      -- sets database and schema
SELECT CURRENT_DATABASE(), CURRENT_SCHEMA();
SELECT * FROM curated.orders;          -- schema.object (uses current database)
SELECT * FROM sales.curated.orders;    -- fully qualified (recommended in pipelines)
```

## 2. Databases

### 2.1 Database Types

| Type | Description | Key Properties |
| --- | --- | --- |
| **Standard (permanent) database** | The default; holds your data | Can contain permanent and transient schemas and tables (retention rules in section 4) |
| **Transient database** | `CREATE TRANSIENT DATABASE` | Everything created inside is transient: lower storage cost, no Fail-safe (section 6) |
| **Shared / imported database** | Created `FROM SHARE` or from a Marketplace / Internal Marketplace listing | **Read-only** to the consumer; no storage copy; privileges managed through `IMPORTED PRIVILEGES` |
| **Application database** | Created when a Native App is installed | Managed by the app and its application roles |
| **Catalog-linked database** | Linked to an external Iceberg REST catalog | Auto-syncs namespaces and Iceberg tables (see the Iceberg guide) |
| **Secondary (replica) database** | Created `AS REPLICA OF` a primary database | Read-only copy for replication and failover; refreshed from the primary |
| **Clone** | `CREATE DATABASE x CLONE y` | Zero-copy snapshot; independent after creation (becomes a normal standard/transient database) |
| **System databases** | `SNOWFLAKE` (usage, governance, security views and Snowflake-provided database roles) and `SNOWFLAKE_SAMPLE_DATA` (sample datasets) | Both are shared databases |

**Note:** There is **no temporary database**. Temporary objects exist at the **table** level (and a few others), not for databases or schemas.

### 2.2 Permanent vs Transient Databases

A transient database makes **everything inside it transient** (all schemas and tables), trading Fail-safe and long Time Travel for lower storage cost. Use it for staging, scratch, and reproducible data. The full permanent, transient, and temporary comparison is in section 6, and retention rules are in section 4.

### 2.3 Database DDL and Operations

```sql
-- Create
CREATE DATABASE sales
  DATA_RETENTION_TIME_IN_DAYS = 14
  COMMENT = 'Sales domain';
CREATE TRANSIENT DATABASE staging_db DATA_RETENTION_TIME_IN_DAYS = 0;
CREATE DATABASE IF NOT EXISTS sales;          -- no error if it exists
CREATE OR REPLACE DATABASE sales_dev CLONE sales;           -- zero-copy clone
CREATE DATABASE sales_yesterday CLONE sales
  AT (TIMESTAMP => DATEADD(day, -1, CURRENT_TIMESTAMP())); -- clone as of the past

-- Alter
ALTER DATABASE sales RENAME TO sales_prod;
ALTER DATABASE sales_prod SWAP WITH sales_dev;              -- atomic name swap
ALTER DATABASE sales_prod SET DATA_RETENTION_TIME_IN_DAYS = 30, COMMENT = 'Prod';
ALTER DATABASE sales_prod UNSET COMMENT;

-- Inspect
SHOW DATABASES;
SHOW DATABASES HISTORY;          -- includes dropped databases within retention
DESCRIBE DATABASE sales_prod;    -- lists schemas

-- Drop and recover
DROP DATABASE sales_dev;         -- drops all contained schemas and objects
UNDROP DATABASE sales_dev;       -- within the Time Travel retention period
```

| Operation | Notes |
| --- | --- |
| **`CREATE OR REPLACE`** | Drops and recreates; the old database and everything inside is gone (recoverable only via Time Travel on the dropped one). Grants on it are lost |
| **`RENAME`** | Updates name only. References by name in views, tasks, and queries can break |
| **`SWAP WITH`** | Atomically exchanges two databases' names and metadata: useful for blue/green releases |
| **`DROP`** | Cascades to all schemas and objects; dropping a database with dependencies (views in other databases, shares) needs care |
| **`UNDROP`** | Restores a dropped database (and its schemas/objects) if still within retention; fails if a database with the same name exists |

### 2.4 Replication, Failover, and Sharing (Overview)

| Feature | How Databases Take Part |
| --- | --- |
| **Replication / failover** | A primary database is replicated to secondary databases in other regions or clouds (via replication or failover groups); secondary databases are read-only until promoted |
| **Secure Data Sharing** | A database is the usual unit exposed to consumers through a share; consumers create a **read-only database from the share**, with no data copy |
| **Database roles** | Roles scoped to the database; can be granted into shares and travel with replication |

### 2.5 Database- and Schema-Level Parameters and Properties

Several **object parameters** can be set on a database or schema and are inherited by objects inside (see the parameters guide for precedence rules):

| Parameter / Property | Levels | Effect |
| --- | --- | --- |
| `DATA_RETENTION_TIME_IN_DAYS` | Account, database, schema, table | Time Travel window |
| `MAX_DATA_EXTENSION_TIME_IN_DAYS` | Account, database, schema, table | How far retention may be extended to keep streams from going stale |
| `DEFAULT_DDL_COLLATION` | Account, database, schema, table | Default collation for new string columns |
| `EXTERNAL_VOLUME`, `CATALOG` (Iceberg defaults) | Account, database, schema | Default Iceberg storage and catalog for new Iceberg tables (confirm in the Iceberg docs) |
| `SUSPEND_TASK_AFTER_NUM_FAILURES` | Account, database, schema, task | Auto-suspend tasks after repeated failures |
| `COMMENT`, **tags** | Database, schema | Documentation and governance labels (owner, domain, classification) |

```sql
ALTER DATABASE sales SET DEFAULT_DDL_COLLATION = 'en-ci';
SHOW PARAMETERS LIKE 'DATA_RETENTION%' IN DATABASE sales;
ALTER SCHEMA sales.curated SET TAG governance.domain = 'sales';
```

## 3. Schemas

### 3.1 Schema Types

| Type | Description | Key Properties |
| --- | --- | --- |
| **Permanent schema** | Default schema type | Can hold permanent, transient, and temporary tables |
| **Transient schema** | `CREATE TRANSIENT SCHEMA` | All tables created inside are transient (see 3.3 and section 6) |
| **Managed access schema** | `WITH MANAGED ACCESS` | Only the **schema owner** or a role with **`MANAGE GRANTS`** can grant privileges on objects in the schema; object owners cannot |
| **`PUBLIC` schema** | Created automatically in every new database | Convenience default; avoid putting real data here |
| **`INFORMATION_SCHEMA`** | Created automatically in every database | **Read-only** views and table functions with metadata about that database's objects (and some account-level metadata) |
| **Shared / replica schemas** | Inside shared or secondary databases | Read-only |
| **Clone** | `CREATE SCHEMA x CLONE y` | Zero-copy snapshot of the schema's objects |

### 3.2 Schema DDL and Operations

```sql
CREATE SCHEMA sales.raw;
CREATE SCHEMA sales.curated WITH MANAGED ACCESS;
CREATE TRANSIENT SCHEMA sales.staging DATA_RETENTION_TIME_IN_DAYS = 0;
CREATE SCHEMA sales.curated_dev CLONE sales.curated;

ALTER SCHEMA sales.raw RENAME TO sales.landing;
ALTER SCHEMA sales.landing SWAP WITH sales.raw_new;
ALTER SCHEMA sales.curated ENABLE MANAGED ACCESS;     -- or DISABLE MANAGED ACCESS
ALTER SCHEMA sales.curated SET DATA_RETENTION_TIME_IN_DAYS = 30;

SHOW SCHEMAS IN DATABASE sales;
DESCRIBE SCHEMA sales.curated;
DROP SCHEMA sales.staging;
UNDROP SCHEMA sales.staging;
```

### 3.3 Transient and Managed Access Rules

| Rule | Detail |
| --- | --- |
| **Transient is inherited downward** | Everything created in a transient database or schema is transient. A permanent table **cannot** be created in a transient schema |
| **Transient cannot be converted** | You cannot alter a permanent schema/database to transient (or the reverse); recreate and copy instead |
| **Managed access centralises grants** | Even the object owner cannot `GRANT` on its own table; only the schema owner or a `MANAGE GRANTS` holder can. Owners keep ownership and can still use the object. `ENABLE` / `DISABLE MANAGED ACCESS` can be switched on an existing schema |

### 3.4 `INFORMATION_SCHEMA` vs `ACCOUNT_USAGE`

| Aspect | `<db>.INFORMATION_SCHEMA` | `SNOWFLAKE.ACCOUNT_USAGE` |
| --- | --- | --- |
| Scope | One database | Whole account |
| Latency | Real time | Up to a few hours for some views |
| History | Shorter retention | Up to 365 days; **includes dropped objects** |
| Examples | `TABLES`, `COLUMNS`, `TABLE_CONSTRAINTS`, `REFERENTIAL_CONSTRAINTS` | `DATABASES`, `SCHEMATA`, `TABLES`, `COLUMNS`, `TABLE_STORAGE_METRICS` |

### 3.5 Dependencies and Storage Monitoring

Before renaming, swapping, or dropping a database or schema, check what depends on it.

```sql
-- Objects that depend on a given object (views on tables, etc.); Account Usage has latency
SELECT referenced_database, referenced_schema, referenced_object_name,
       referencing_database, referencing_schema, referencing_object_name, referencing_object_domain
FROM snowflake.account_usage.object_dependencies
WHERE referenced_database = 'SALES'
  AND referencing_database <> 'SALES';          -- cross-database dependants

-- Storage by database over time and by table
SELECT usage_date, database_name, average_database_bytes, average_failsafe_bytes
FROM snowflake.account_usage.database_storage_usage_history
WHERE usage_date >= DATEADD(day, -30, CURRENT_DATE())
ORDER BY usage_date DESC, average_database_bytes DESC;

SELECT table_catalog, table_schema, table_name, active_bytes, time_travel_bytes, failsafe_bytes
FROM snowflake.account_usage.table_storage_metrics
WHERE table_catalog = 'SALES'
ORDER BY active_bytes DESC;
```

| View | Use |
| --- | --- |
| `OBJECT_DEPENDENCIES` | Impact analysis for renames, swaps, drops, and column changes |
| `DATABASES`, `SCHEMATA` | Inventory, creation and deletion times, owners, retention |
| `DATABASE_STORAGE_USAGE_HISTORY` | Daily database storage and Fail-safe bytes |
| `TABLE_STORAGE_METRICS` | Active, Time Travel, Fail-safe, and retained-for-clone bytes per table |

## 4. Time Travel, Fail-safe, and Retention (Single Reference)

| Feature | Detail |
| --- | --- |
| **Time Travel** | Query, clone, or restore data, and recover dropped objects, within `DATA_RETENTION_TIME_IN_DAYS`. Maximum retention depends on the **table type and edition** (table in section 6.1) |
| **Fail-safe** | Fixed **7-day** last-resort recovery after Time Travel ends; **permanent tables only**; not configurable and not self-service (Snowflake Support performs recovery) |
| **Retention hierarchy** | Account, Database, Schema, Table: the most specific explicit value wins. Account-level `MIN_DATA_RETENTION_TIME_IN_DAYS` is a floor (see the parameters guide) |
| **`UNDROP`** | Works for databases, schemas, and tables within retention; fails if an object with the same name already exists |
| **Retention of 0** | Disables Time Travel and `UNDROP`. A **permanent** table still has Fail-safe, but you cannot self-recover from it |
| **Storage billing** | Active data + Time Travel data + Fail-safe data, all measured compressed |

## 5. Zero-Copy Cloning of Databases and Schemas

* A clone is a **metadata-only snapshot**: it points to the same micro-partitions and consumes **no extra storage** until data changes in either copy.
* After cloning, source and clone are **independent**; changes do not propagate.
* Clone at database, schema, or table level, optionally **as of a past time** (`AT` / `BEFORE`, within retention).
* **Common uses:** instant dev/test copies, safe release testing, point-in-time snapshots, backups before risky changes.
* **Behaviours to remember** (confirm details in the cloning docs): tasks in a clone start **suspended**; privileges granted on **child objects** are retained but privileges on the cloned container itself are not copied; some object types (for example, certain stages and pipes) have special handling; clones of shared databases are not allowed.
* **Clone type rules** by table type are in section 6.3.

## 6. Tables: Permanent, Transient, and Temporary (Plus Other Types)

### 6.1 The Three Core Table Types

| Aspect | **Permanent** | **Transient** | **Temporary** |
| --- | --- | --- | --- |
| **Create with** | `CREATE TABLE` | `CREATE TRANSIENT TABLE` | `CREATE TEMPORARY TABLE` (synonyms: `TEMP`, `LOCAL TEMP`, `GLOBAL TEMP`, `VOLATILE`; all are session-scoped) |
| **Lifetime** | Until dropped | Until dropped | **Until the creating session ends** (or dropped earlier) |
| **Visibility** | Any session with privileges | Any session with privileges | **Only the creating session**: not other sessions, tasks, or users, even with grants |
| **Time Travel (Standard edition)** | 0 or 1 day | 0 or 1 day | 0 or 1 day |
| **Time Travel (Enterprise and above)** | **0 to 90 days** | 0 or 1 day | 0 or 1 day |
| **Fail-safe** | **7 days** | **None** | **None** |
| **Recovery after drop** | `UNDROP` within retention, then Fail-safe via Snowflake Support | `UNDROP` within retention (at most 1 day) | Not recoverable once the session has ended |
| **Storage billed for** | Active + Time Travel + Fail-safe | Active + Time Travel (at most 1 day) | Active + Time Travel (at most 1 day), only while the session lives |
| **Allowed in** | Permanent schemas and databases only | Permanent or transient schemas | Any schema |
| **Constraints** | Supported | Supported | Supported |
| **Typical use** | Sources of truth, curated and compliance data | Staging and landing data, reproducible layers, dev copies | Intermediate results within one job or session |

* **Constraints:** `PRIMARY KEY`, `UNIQUE`, and `FOREIGN KEY` are informational on all three; `NOT NULL` and `CHECK` are enforced (see section 7).
* **Containers decide the type:** a table created in a transient schema or database is transient (see 3.3), so a permanent table cannot live there.
* **Name shadowing:** a temporary table with the same name as a permanent or transient table in the same schema **hides** that table for the creating session only, which can silently redirect queries.

### 6.2 Creating the Three Types

```sql
-- Permanent: default
CREATE TABLE orders (
  order_id NUMBER NOT NULL, customer_id NUMBER, order_ts TIMESTAMP_NTZ, amount NUMBER(12,2)
) DATA_RETENTION_TIME_IN_DAYS = 14;

-- Transient: no Fail-safe, short Time Travel
CREATE TRANSIENT TABLE stg_orders (
  order_id NUMBER, customer_id NUMBER, order_ts TIMESTAMP_NTZ, amount NUMBER(12,2)
) DATA_RETENTION_TIME_IN_DAYS = 0;

-- Temporary: session-scoped scratch table
CREATE TEMPORARY TABLE tmp_daily_totals AS
SELECT customer_id, SUM(amount) AS total
FROM orders
GROUP BY customer_id;

-- Change retention later (cannot exceed the type's maximum)
ALTER TABLE orders SET DATA_RETENTION_TIME_IN_DAYS = 30;

DROP TABLE tmp_daily_totals;   -- release storage early instead of waiting for session end
```

### 6.3 Converting and Cloning Between Types

There is **no `ALTER TABLE` that changes a table's type**. To change type, create a new table and copy or swap.

```sql
-- Promote staging data into a permanent table
CREATE TABLE orders_curated AS SELECT * FROM stg_orders;      -- new table is permanent (in a permanent schema)

-- Atomic cutover to the new table
ALTER TABLE orders_curated SWAP WITH orders;

-- Cheap dev copy of a permanent table as transient
CREATE TRANSIENT TABLE orders_dev CLONE orders;
```

| Rule | Detail |
| --- | --- |
| **CTAS / `LIKE`** | The new table's type comes from the keyword you use (and from a transient container) |
| **Cloning** | A transient or temporary table **cannot** be cloned as permanent. A permanent table can be cloned as transient or temporary (confirm exact combinations in the cloning docs) |
| **Clone at a past time** | `CLONE ... AT` or `BEFORE` works within the retention period, so a transient table with 0 days cannot be cloned as of the past |
| **Schemas and databases** | Same idea: a transient schema or database cannot be altered into permanent (see 3.3) |

### 6.4 Temporary Table Behaviour in Practice

| Topic | Detail |
| --- | --- |
| **Scope** | Tied to one session. A new connection, a new worksheet session, or a different user sees nothing |
| **Tasks and other jobs** | A task runs in its own session, so it cannot see a temporary table created elsewhere |
| **Stored procedures** | They run in the caller's session, so a temporary table they create normally stays until the session ends or you drop it |
| **Connection pools** | Pooled sessions are reused, so a temporary table can outlive a "request" and collide with the next user of that session: use unique names and drop explicitly |
| **Cost** | Storage is billed while the table exists; a client that disconnects without closing the session cleanly can leave data lingering until the session expires |
| **Snowpark** | `DataFrame.cache_result()` materializes results in a temporary table |
| **Not for** | Anything another session, task, or person must read, or anything that must survive the session |

### 6.5 Choosing the Right Type

| Data | Recommended Type | Retention |
| --- | --- | --- |
| Business source of truth, financial or regulated data | **Permanent** | Set per SLA (up to 90 days on Enterprise and above) |
| Re-loadable landing/staging data from files or streams | **Transient** | 0 or 1 day |
| High-churn intermediate layers that pipelines rebuild | **Transient** | 0 or 1 day (avoids Fail-safe storage on rewritten data) |
| Dev and test copies of production | **Transient** clone | 0 or 1 day |
| Step-by-step intermediate results inside one job | **Temporary** | Default, dropped when done |
| Curated marts consumed by BI | **Permanent** | Moderate (for example, 7 to 14 days) |

```
 Must another session, task, or user read it, or must it survive the session?
   │ No  ──► TEMPORARY
   │ Yes ──► Can you rebuild it from upstream sources if it is lost or corrupted?
               │ Yes ──► TRANSIENT  (lower cost, 0-1 day Time Travel, no Fail-safe)
               │ No  ──► PERMANENT  (Time Travel + 7-day Fail-safe)
```

**Trade-off to remember:** transient and temporary tables have no Fail-safe, so an accidental drop or overwrite is **unrecoverable** once Time Travel (at most 1 day) has passed.

### 6.6 Finding Table Types and Their Cost

```sql
-- Table type and retention in one database
SELECT table_schema, table_name, table_type, is_transient, retention_time
FROM sales.INFORMATION_SCHEMA.TABLES
WHERE table_schema <> 'INFORMATION_SCHEMA'
ORDER BY table_schema, table_name;

SHOW TABLES IN SCHEMA sales.curated;     -- check the kind and retention_time columns

-- Where Fail-safe and Time Travel storage goes, split by table type
SELECT is_transient,
       ROUND(SUM(active_bytes)     / POW(1024,4), 3) AS active_tb,
       ROUND(SUM(time_travel_bytes)/ POW(1024,4), 3) AS time_travel_tb,
       ROUND(SUM(failsafe_bytes)   / POW(1024,4), 3) AS failsafe_tb
FROM snowflake.account_usage.table_storage_metrics
WHERE deleted = FALSE
GROUP BY is_transient;
```

Temporary tables appear only for the session that created them. Review large permanent tables with high churn: if they are reproducible, converting them to transient (create, copy, swap) can remove their Fail-safe storage.

### 6.7 Other Table Types

| Table Type | Persistence | Constraints (PK / UNIQUE / FK) | Notes |
| --- | --- | --- | --- |
| **External** | Data stays in external storage | Not applicable (read-only) | Often superseded by Iceberg or Delta Direct |
| **Hybrid** | Until dropped | **Enforced** (PK required) | Row-store for transactional (OLTP-style) workloads; check current docs for Time Travel and Fail-safe |
| **Dynamic** | Until dropped | Informational | Declarative, auto-refreshed; can be created transient to cut storage cost |
| **Iceberg** | Until dropped | Informational; `CHECK` on Snowflake-managed | Open table format; Fail-safe only on Snowflake storage (see the Iceberg guide) |
| **Event table** | Until dropped | n/a | Telemetry (logs, traces) |

<br></br>

## 7. Constraints and Referential Integrity

### 7.1 Enforcement Matrix

| Constraint | **Standard tables** (permanent, transient, temporary) | **Hybrid tables** |
| --- | --- | --- |
| `NOT NULL` | **Enforced** | Enforced |
| `CHECK` | **Enforced** (where supported) | Check the current hybrid-table docs |
| `PRIMARY KEY` | **Not enforced** (informational metadata) | **Enforced**, and **required** on every hybrid table |
| `UNIQUE` | **Not enforced** | **Enforced** |
| `FOREIGN KEY` | **Not enforced** | **Enforced** (referenced tables must be in the **same database**) |

### 7.2 Constraint Properties

| Property | Meaning for Standard Tables | Meaning for Hybrid Tables |
| --- | --- | --- |
| **`ENFORCED` / `NOT ENFORCED`** | Properties are accepted for compatibility, but PK/UNIQUE/FK are **not** actually enforced | `NOT ENFORCED` **cannot** be set on PK, FK, UNIQUE |
| **`VALIDATE` / `NOVALIDATE`** | Records whether existing data was validated; default for PK/FK is **`NOVALIDATE`**. Specifying `ENABLE` or `VALIDATE` (non-default) when **creating** a PK/UNIQUE/FK on a standard table causes the constraint **not to be created** | Adding UNIQUE/FK via `ALTER TABLE` **always validates** existing rows |
| **`RELY` / `NORELY`** | If you have **guaranteed** the data satisfies the constraint, set `RELY` so the optimizer may use it (Snowflake describes this as enabling query rewrites such as eliminating unnecessary joins); set it on **both** the related PK and FK | Not meaningful: constraints are enforced |
| **`ENABLE` / `DISABLE`** | Accepted with the properties above | n/a |
| **FK referential actions** (`ON UPDATE`, `ON DELETE`) and `MATCH` | Stored as metadata only | Applied for enforced foreign keys (confirm supported actions in the current docs) |

**Warning on `RELY`:** if you set `RELY` but the data **violates** the constraint, queries that depend on the optimizer's assumption can return **wrong results** (for example, a join eliminated because Snowflake trusts a key that is not actually unique). Use it only when integrity is guaranteed.

### 7.3 Syntax Examples

```sql
-- Inline and out-of-line constraints on a standard table
CREATE TABLE customers (
  customer_id  NUMBER        NOT NULL,
  email        STRING        NOT NULL,
  status       STRING        DEFAULT 'ACTIVE',
  age          NUMBER,
  CONSTRAINT pk_customers  PRIMARY KEY (customer_id),
  CONSTRAINT uq_cust_email UNIQUE (email),
  CONSTRAINT ck_cust_age   CHECK (age IS NULL OR age >= 0)       -- enforced
);

CREATE TABLE orders (
  order_id     NUMBER NOT NULL PRIMARY KEY,
  customer_id  NUMBER NOT NULL,
  order_ts     TIMESTAMP_NTZ,
  amount       NUMBER(12,2),
  CONSTRAINT fk_orders_cust FOREIGN KEY (customer_id)
    REFERENCES customers (customer_id)                           -- informational on standard tables
);

-- Tell the optimizer it may trust the keys (only if the data truly complies)
ALTER TABLE customers ALTER CONSTRAINT pk_customers RELY;
ALTER TABLE orders    ALTER CONSTRAINT fk_orders_cust RELY;

-- Add and drop
ALTER TABLE orders ADD CONSTRAINT ck_amount CHECK (amount >= 0);
ALTER TABLE orders DROP CONSTRAINT fk_orders_cust;
ALTER TABLE orders DROP PRIMARY KEY;

-- Hybrid table: constraints ARE enforced, primary key is mandatory
CREATE HYBRID TABLE accounts (
  account_id  NUMBER PRIMARY KEY,
  owner_email STRING UNIQUE,
  balance     NUMBER(18,2) NOT NULL
);
```

### 7.4 Inspecting Constraints

```sql
SHOW PRIMARY KEYS  IN TABLE customers;
SHOW UNIQUE KEYS   IN TABLE customers;
SHOW IMPORTED KEYS IN TABLE orders;        -- foreign keys this table has (child side)
SHOW EXPORTED KEYS IN TABLE customers;     -- foreign keys that reference this table (parent side)

SELECT * FROM sales.INFORMATION_SCHEMA.TABLE_CONSTRAINTS WHERE table_name = 'ORDERS';
SELECT * FROM sales.INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS;
```

`SHOW IMPORTED KEYS` and `SHOW EXPORTED KEYS` are widely used by BI and data modelling tools to draw relationships.

### 7.5 Why Declare Informational Constraints?

| Reason | Detail |
| --- | --- |
| **Documentation and ERDs** | Modelling and catalog tools read them to show relationships |
| **BI tool behaviour** | Some BI tools use PK/FK metadata to auto-join or validate models |
| **Optimizer hints** | `RELY` lets the optimizer use them (see 7.2) |
| **Migration compatibility** | DDL from other databases loads without modification |
| **Contract** | Communicates intended grain and keys to engineers |

### 7.6 Hybrid Tables and Referential Integrity

Enforcement rules (primary key required, same-database foreign keys, no `NOT ENFORCED`, validation on `ALTER`) are in 7.1 and 7.2.

| Point | Detail |
| --- | --- |
| **Typical use** | Transactional, low-latency, row-level lookups and updates where integrity must be guaranteed |
| **Trade-offs** | Different storage and performance profile from standard tables; check the current hybrid-table limitations before choosing |

### 7.7 Constraints Across Databases and Table Types

* Foreign keys on **standard** tables may point to tables in other schemas or databases (they are only metadata).
* Constraints are supported on permanent, transient, temporary, and hybrid tables; **not** on external tables.
* `CHECK` constraints are limited to standard tables and Snowflake-managed Iceberg tables.
* Cloning a table clones its constraint definitions.

### 7.8 Keys and Generated Values

| Mechanism | Guarantee | Caveat |
| --- | --- | --- |
| **`AUTOINCREMENT` / `IDENTITY`** | Generates numbers that are unique within that column's sequence | **Not gap-free and not guaranteed ordered**, especially with parallel loads; don't use it to infer load order |
| **`SEQUENCE`** | Unique values from `NEXTVAL`; shareable across tables | Gaps are normal (failed statements, caching); not a business key |
| **`UUID_STRING()`** | Practically unique random identifiers | Wide, non-sortable keys; fine for surrogate keys |
| **Hash keys** (e.g., `SHA2`/`MD5` of business keys) | Deterministic, reproducible across loads | Collisions are theoretically possible; keep the business key too |
| **Natural / business keys** | Meaningful and stable | Must be truly unique; verify with checks since uniqueness isn't enforced on standard tables |

**Composite and self-referencing keys** are allowed (for example, `PRIMARY KEY (order_id, line_no)` or an `employee.manager_id` foreign key referencing `employee.employee_id`); like other FKs they are informational on standard tables.

## 8. Identifiers and Naming

| Rule | Detail |
| --- | --- |
| **Unquoted identifiers** | Start with a letter or underscore; may contain letters, digits, underscores, and `$`; stored and resolved as **UPPERCASE**, so case-insensitive |
| **Quoted identifiers** | `"My Table"`: can contain spaces and special characters; **case-sensitive** and must always be quoted |
| **Length** | Up to **255** characters |
| **Reserved words** | Cannot be used unquoted as identifiers |
| **`QUOTED_IDENTIFIERS_IGNORE_CASE`** | Session/account parameter that makes quoted identifiers case-insensitive (default `FALSE`) |
| **Uniqueness** | Names are unique **per container**: database names per account, schema names per database, object names per schema (and type rules apply) |
| **Convention advice** | Use unquoted `UPPER_SNAKE_CASE`, avoid quoted mixed-case names |

## 9. Access Control on Databases and Schemas

| Object | Key Privileges |
| --- | --- |
| **Database** | `USAGE`, `MONITOR`, `MODIFY`, `CREATE SCHEMA`, `CREATE DATABASE ROLE`, `IMPORTED PRIVILEGES` (shared databases), `OWNERSHIP` |
| **Schema** | `USAGE`, `MONITOR`, `MODIFY`, `CREATE <object type>` (table, view, stage, task, ...), `OWNERSHIP` |

* To use any object you need `USAGE` on its **database** and **schema** plus the object privilege.
* `FUTURE` and `ALL` grants at schema or database level cover new and existing objects.
* Dropping or replacing a database/schema requires `OWNERSHIP`.

## 10. Quick Interview Q&A

| Question | Short Answer |
| --- | --- |
| Snowflake object hierarchy? | Organization, account, database, schema, then objects such as tables and views. |
| Types of databases? | Standard (permanent), transient, shared/imported (read-only), application, catalog-linked, secondary (replica), and clones; `SNOWFLAKE` and `SNOWFLAKE_SAMPLE_DATA` are system shared databases. There is no temporary database. |
| Permanent vs transient vs temporary tables? | Permanent: up to 90 days Time Travel (Enterprise+) plus 7-day Fail-safe. Transient: persists until dropped, 0 or 1 day Time Travel, no Fail-safe. Temporary: visible only to the creating session, dropped when it ends, 0 or 1 day Time Travel, no Fail-safe. |
| Types of schemas? | Permanent, transient, managed access, plus the automatic `PUBLIC` and `INFORMATION_SCHEMA`. |
| What is a managed access schema? | A schema where only the schema owner or a `MANAGE GRANTS` role can grant privileges on its objects. |
| What does `INFORMATION_SCHEMA` provide? | Read-only metadata views and table functions for that database, in real time. |
| Are constraints enforced in Snowflake? | Only `NOT NULL` and `CHECK` on standard tables. `PRIMARY KEY`, `UNIQUE`, and `FOREIGN KEY` are informational on standard tables and enforced on hybrid tables. |
| Why define a PK/FK if not enforced? | Documentation, BI/modelling tools, optimizer hints with `RELY`, and migration compatibility. |
| What does `RELY` do? | Tells the optimizer it may trust the constraint (e.g., for join elimination); wrong data under `RELY` can produce wrong results. |
| How do you prevent duplicates on a standard table? | `MERGE` on the business key, dedupe with `QUALIFY ROW_NUMBER()`, validation queries, data metric functions, and transform-layer tests. |
| What is required for a hybrid table? | A primary key; PK/UNIQUE/FK are enforced, and FKs must reference tables in the same database. |
| What does a clone do? | Zero-copy snapshot of metadata; independent afterwards; can be created as of a past time within retention. |
| `SWAP WITH` use? | Atomically exchange the names of two databases or schemas (e.g., blue/green releases). |
| How do you recover a dropped database? | `UNDROP DATABASE` within the retention period; after that, only Snowflake Fail-safe (permanent objects). |
| How are retention parameters inherited? | Account, database, schema, table: the most specific explicit value applies; the account minimum acts as a floor. |
| What permissions are needed to read a table? | `USAGE` on database and schema, `SELECT` on the table, and `USAGE` on a warehouse. |
| How do unquoted identifiers behave? | They are case-insensitive and stored in uppercase; quoted identifiers are case-sensitive. |
| Are AUTOINCREMENT values gap-free? | No. They are unique but can have gaps and are not guaranteed to be ordered, particularly with parallel loads. |
| How do you check what depends on a database before dropping it? | Query `ACCOUNT_USAGE.OBJECT_DEPENDENCIES` (and review shares, tasks, and cross-database views). |
| Can you clone a transient or temporary table as permanent? | No. A permanent table can be cloned as transient or temporary; confirm exact combinations in the cloning docs. |
| How do you turn a transient table into a permanent one? | You cannot alter its type: create a permanent table with CTAS (in a permanent schema), then swap or rename. |
| Who can see a temporary table? | Only the session that created it; it is dropped when that session ends and cannot be recovered afterwards. |
| What happens if a temporary table has the same name as a permanent one? | Within that session, the temporary table hides the permanent one. |
| What is the maximum Time Travel for transient and temporary tables? | 1 day, even on Enterprise edition and above. |
| Does a permanent table with retention 0 have Fail-safe? | Yes, 7 days, but recovery is only through Snowflake Support. |
| What storage does a permanent table incur beyond active data? | Time Travel data (within retention) and Fail-safe data (7 days). |
| How do you find transient tables and their retention? | `INFORMATION_SCHEMA.TABLES` or `ACCOUNT_USAGE.TABLES` (`IS_TRANSIENT`, `RETENTION_TIME`), or `SHOW TABLES`. |
