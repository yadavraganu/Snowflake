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

<br></br>

## 2. Databases

### 2.1 Database Types

| Type | Description | Key Properties |
| --- | --- | --- |
| **Standard (permanent) database** | The default; holds your data | Time Travel up to 90 days (Enterprise+), **Fail-safe 7 days** for permanent tables |
| **Transient database** | `CREATE TRANSIENT DATABASE` | Time Travel **0 or 1 day**, **no Fail-safe**; all schemas and tables created inside are transient; lower storage cost |
| **Shared / imported database** | Created `FROM SHARE` or from a Marketplace / Internal Marketplace listing | **Read-only** to the consumer; no storage copy; privileges managed through `IMPORTED PRIVILEGES` |
| **Application database** | Created when a Native App is installed | Managed by the app and its application roles |
| **Catalog-linked database** | Linked to an external Iceberg REST catalog | Auto-syncs namespaces and Iceberg tables (see the Iceberg guide) |
| **Secondary (replica) database** | Created `AS REPLICA OF` a primary database | Read-only copy for replication and failover; refreshed from the primary |
| **Clone** | `CREATE DATABASE x CLONE y` | Zero-copy snapshot; independent after creation (becomes a normal standard/transient database) |
| **System databases** | `SNOWFLAKE` (usage, governance, security views and Snowflake-provided database roles) and `SNOWFLAKE_SAMPLE_DATA` (sample datasets) | Both are shared databases |

**Note:** There is **no temporary database**. Temporary objects exist at the **table** level (and a few others), not for databases or schemas.

### 2.2 Permanent vs Transient

| Aspect | Permanent | Transient |
| --- | --- | --- |
| Time Travel | 0 to 1 day (Standard); up to **90 days** (Enterprise and above) | **0 or 1 day** |
| Fail-safe | **7 days** | **None** |
| Storage cost | Includes Time Travel and Fail-safe storage | Lower: no Fail-safe storage |
| Typical use | Production, curated, critical data | Staging, scratch, easily reproducible data |
| Child objects | Can contain permanent **and** transient schemas/tables | **All** children are transient |

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
| **Retention** | `DATA_RETENTION_TIME_IN_DAYS` at database level is **inherited** by schemas and tables without their own value (see the parameters guide) |

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

<br></br>

## 3. Schemas

### 3.1 Schema Types

| Type | Description | Key Properties |
| --- | --- | --- |
| **Permanent schema** | Default schema type | Contains permanent (and optionally transient) tables; retention and Fail-safe as per the table type |
| **Transient schema** | `CREATE TRANSIENT SCHEMA` | All tables created inside are transient: Time Travel 0 or 1 day, **no Fail-safe** |
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
| **Managed access centralises grants** | Even the object owner cannot `GRANT` on its own table in a managed access schema; the schema owner or `MANAGE GRANTS` holder does it |
| **Managed access and ownership** | Owners keep ownership and can still use the object; only the grant ability moves |
| **Switching managed access** | `ENABLE` or `DISABLE` MANAGED ACCESS on an existing schema is allowed |

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

<br></br>

## 4. Data Protection and Retention at Database/Schema Level

| Feature | Detail |
| --- | --- |
| **Time Travel** | Query, clone, or restore data and dropped objects within `DATA_RETENTION_TIME_IN_DAYS` |
| **Retention hierarchy** | Account, Database, Schema, Table: the most specific explicit value wins; `MIN_DATA_RETENTION_TIME_IN_DAYS` (account) is a floor |
| **Fail-safe** | 7-day last-resort recovery after Time Travel, **permanent** objects only |
| **UNDROP** | Available for databases, schemas, and tables (and some other objects) within retention |
| **Cloning in time** | `CLONE ... AT` or `CLONE ... BEFORE` creates a copy as of a past point within retention |

<br></br>

## 5. Zero-Copy Cloning of Databases and Schemas

* A clone is a **metadata-only snapshot**: it points to the same micro-partitions and consumes **no extra storage** until data changes in either copy.
* After cloning, source and clone are **independent**; changes do not propagate.
* Clone at database, schema, or table level, optionally **as of a past time** (`AT` / `BEFORE`, within retention).
* **Common uses:** instant dev/test copies, safe release testing, point-in-time snapshots, backups before risky changes.
* **Behaviours to remember** (confirm details in the cloning docs): tasks in a clone start **suspended**; privileges granted on **child objects** are retained but privileges on the cloned container itself are not copied; some object types (for example, certain stages and pipes) have special handling; clones of shared databases are not allowed.
* **Clone type rule (confirm in the cloning docs):** a transient object can only be cloned as transient (or temporary, for tables), never as permanent. A permanent object can typically be cloned as transient (for example, `CREATE TRANSIENT DATABASE ... CLONE`) to save Time Travel and Fail-safe storage on dev copies.

<br></br>

## 6. Table Types (Context for Constraints and Retention)

| Table Type | Persistence | Time Travel | Fail-safe | Constraints (PK / UNIQUE / FK) | Notes |
| --- | --- | --- | --- | --- | --- |
| **Permanent** (standard) | Until dropped | Up to 90 days (Enterprise+) | 7 days | Informational | Default |
| **Transient** | Until dropped | 0 or 1 day | None | Informational | Staging / reproducible data |
| **Temporary** | **Session only** | 0 or 1 day | None | Informational | Visible only to the creating session; dropped at session end |
| **External** | Data stays in external storage | None | None | Not applicable (read-only) | Superseded in many cases by Iceberg / Delta Direct |
| **Hybrid** | Until dropped | See current docs | See current docs | **Enforced** (PK required) | Row-store for transactional (OLTP-style) workloads |
| **Dynamic** | Until dropped | As permanent/transient | As permanent/transient | Informational | Declarative, auto-refreshed pipelines |
| **Iceberg** | Until dropped | Retention-based | Only on Snowflake storage | Informational; `CHECK` on Snowflake-managed | Open table format (see the Iceberg guide) |
| **Event table** | Until dropped | Per settings | Per settings | n/a | Telemetry (logs, traces) |

Constraint support is available on permanent, transient, temporary, and hybrid tables; **only hybrid tables enforce `PRIMARY KEY`, `UNIQUE`, and `FOREIGN KEY`**.

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

### 7.3 Constraint Properties

| Property | Meaning for Standard Tables | Meaning for Hybrid Tables |
| --- | --- | --- |
| **`ENFORCED` / `NOT ENFORCED`** | Properties are accepted for compatibility, but PK/UNIQUE/FK are **not** actually enforced | `NOT ENFORCED` **cannot** be set on PK, FK, UNIQUE |
| **`VALIDATE` / `NOVALIDATE`** | Records whether existing data was validated; default for PK/FK is **`NOVALIDATE`**. Specifying `ENABLE` or `VALIDATE` (non-default) when **creating** a PK/UNIQUE/FK on a standard table causes the constraint **not to be created** | Adding UNIQUE/FK via `ALTER TABLE` **always validates** existing rows |
| **`RELY` / `NORELY`** | If you have **guaranteed** the data satisfies the constraint, set `RELY` so the optimizer may use it (Snowflake describes this as enabling query rewrites such as eliminating unnecessary joins); set it on **both** the related PK and FK | Not meaningful: constraints are enforced |
| **`ENABLE` / `DISABLE`** | Accepted with the properties above | n/a |
| **FK referential actions** (`ON UPDATE`, `ON DELETE`) and `MATCH` | Stored as metadata only | Applied for enforced foreign keys (confirm supported actions in the current docs) |

**Warning on `RELY`:** if you set `RELY` but the data **violates** the constraint, queries that depend on the optimizer's assumption can return **wrong results** (for example, a join eliminated because Snowflake trusts a key that is not actually unique). Use it only when integrity is guaranteed.

### 7.4 Syntax Examples

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

### 7.5 Inspecting Constraints

```sql
SHOW PRIMARY KEYS  IN TABLE customers;
SHOW UNIQUE KEYS   IN TABLE customers;
SHOW IMPORTED KEYS IN TABLE orders;        -- foreign keys this table has (child side)
SHOW EXPORTED KEYS IN TABLE customers;     -- foreign keys that reference this table (parent side)

SELECT * FROM sales.INFORMATION_SCHEMA.TABLE_CONSTRAINTS WHERE table_name = 'ORDERS';
SELECT * FROM sales.INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS;
```

`SHOW IMPORTED KEYS` and `SHOW EXPORTED KEYS` are widely used by BI and data modelling tools to draw relationships.

### 7.6 Why Declare Informational Constraints?

| Reason | Detail |
| --- | --- |
| **Documentation and ERDs** | Modelling and catalog tools read them to show relationships |
| **BI tool behaviour** | Some BI tools use PK/FK metadata to auto-join or validate models |
| **Optimizer hints** | With `RELY`, join elimination and similar optimizations become possible |
| **Migration compatibility** | DDL from other databases loads without modification |
| **Contract** | Communicates intended grain and keys to engineers |

### 7.7 Hybrid Tables and Referential Integrity

| Point | Detail |
| --- | --- |
| **Primary key required** | Every hybrid table needs one |
| **Enforced** | PK, UNIQUE, and FK violations raise errors at DML time |
| **Same-database FKs** | Foreign keys between hybrid tables must reference tables in the **same database** |
| **No `NOT ENFORCED`** | Cannot be turned off for PK/FK/UNIQUE |
| **`ALTER TABLE` adds validate data** | Adding UNIQUE or FK validates existing rows |
| **Typical use** | Transactional, low-latency, row-level lookups and updates where integrity must be guaranteed |
| **Trade-offs** | Different storage and performance profile from standard tables; check the current hybrid-table limitations before choosing |

### 7.8 Constraints Across Databases and Table Types

* Foreign keys on **standard** tables may point to tables in other schemas or databases (they are only metadata).
* Constraints are supported on permanent, transient, temporary, and hybrid tables; **not** on external tables.
* `CHECK` constraints are limited to standard tables and Snowflake-managed Iceberg tables.
* Cloning a table clones its constraint definitions.

### 7.9 Keys and Generated Values

| Mechanism | Guarantee | Caveat |
| --- | --- | --- |
| **`AUTOINCREMENT` / `IDENTITY`** | Generates numbers that are unique within that column's sequence | **Not gap-free and not guaranteed ordered**, especially with parallel loads; don't use it to infer load order |
| **`SEQUENCE`** | Unique values from `NEXTVAL`; shareable across tables | Gaps are normal (failed statements, caching); not a business key |
| **`UUID_STRING()`** | Practically unique random identifiers | Wide, non-sortable keys; fine for surrogate keys |
| **Hash keys** (e.g., `SHA2`/`MD5` of business keys) | Deterministic, reproducible across loads | Collisions are theoretically possible; keep the business key too |
| **Natural / business keys** | Meaningful and stable | Must be truly unique; verify with checks since uniqueness isn't enforced on standard tables |

**Composite and self-referencing keys** are allowed (for example, `PRIMARY KEY (order_id, line_no)` or an `employee.manager_id` foreign key referencing `employee.employee_id`); like other FKs they are informational on standard tables.

<br></br>

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

<br></br>

## 9. Access Control on Databases and Schemas

| Object | Key Privileges |
| --- | --- |
| **Database** | `USAGE`, `MONITOR`, `MODIFY`, `CREATE SCHEMA`, `CREATE DATABASE ROLE`, `IMPORTED PRIVILEGES` (shared databases), `OWNERSHIP` |
| **Schema** | `USAGE`, `MONITOR`, `MODIFY`, `CREATE <object type>` (table, view, stage, task, ...), `OWNERSHIP` |

* To use any object you need `USAGE` on its **database** and **schema** plus the object privilege.
* `FUTURE` and `ALL` grants at schema or database level cover new and existing objects.
* **Managed access schemas** centralise grant control (see the roles guide).
* Dropping or replacing a database/schema requires `OWNERSHIP`.

<br></br>

## 10. Design Patterns and Best Practices

### 10.1 Environment and Domain Layout

```
 Option A: database per environment             Option B: database per domain + env
 ┌ RAW_DB      ┐  ┌ STAGING_DB ┐  ┌ PROD_DB ┐    ┌ PROD_SALES ┐  ┌ PROD_FINANCE ┐
 │ schemas by  │  │ transient  │  │ curated │    │ RAW        │  │ RAW          │
 │ source      │  │            │  │ marts   │    │ CURATED    │  │ CURATED      │
 └─────────────┘  └────────────┘  └─────────┘    │ MART       │  │ MART         │
                                                  └────────────┘  └──────────────┘
 DEV and TEST: zero-copy CLONES of production databases
```

### 10.2 Best Practices

* **Separate environments** (DEV/TEST/PROD) by database, and use **clones** to refresh lower environments.
* Use **transient** databases/schemas for staging and reproducible data to cut Time Travel and Fail-safe cost.
* Use **managed access schemas** for curated and shared layers.
* Set **retention** deliberately per layer (see the parameters guide) instead of leaving the default everywhere.
* Keep **business logic and integrity checks in the pipeline** since PK/FK on standard tables are informational.
* Declare **PK/FK for documentation and tools**, and use `RELY` only when integrity is guaranteed.
* Use **fully qualified names** in scripts and pipelines; avoid relying on session context.
* Keep the **`PUBLIC` schema empty**, and avoid granting to `PUBLIC`.
* Use **`SWAP WITH`** for atomic cutovers (blue/green schema or database swaps).
* Use **`UNDROP`** and **Time Travel clones** as safety nets, and know the retention window.
* Apply **tags and comments** on databases and schemas (owner, domain, data classification) for governance.
* Grant access through **roles** (database roles for per-database tiers), never directly to users.

<br></br>

## 11. Common Pitfalls

* Assuming **PRIMARY KEY / FOREIGN KEY / UNIQUE** are enforced on standard tables (they are not).
* Setting **`RELY`** on constraints the data violates, causing **wrong query results**.
* Using **`CREATE OR REPLACE DATABASE/SCHEMA`** in production scripts and losing data and grants.
* Creating a **transient** database for data that actually needs Fail-safe.
* Trying to put a **permanent table in a transient schema**, or expecting to convert transient to permanent.
* Granting `SELECT` but forgetting **`USAGE` on database and schema**.
* Renaming or swapping objects without checking **dependent views, tasks, and pipes**.
* Forgetting that **tasks in clones start suspended**.
* Leaving **`PUBLIC`** schema as the default dumping ground.
* Using **quoted mixed-case identifiers** and then writing unquoted queries that cannot find them.
* Expecting an **`UNDROP`** after the retention period has passed (Fail-safe is not self-service).
* Believing a **clone** is a backup that stays in sync (it is an independent snapshot).

<br></br>

## 12. Quick Interview Q&A

| Question | Short Answer |
| --- | --- |
| Snowflake object hierarchy? | Organization, account, database, schema, then objects such as tables and views. |
| Types of databases? | Standard (permanent), transient, shared/imported (read-only), application, catalog-linked, secondary (replica), and clones; `SNOWFLAKE` and `SNOWFLAKE_SAMPLE_DATA` are system shared databases. There is no temporary database. |
| Transient vs permanent database? | Transient has Time Travel of 0 or 1 day and no Fail-safe, with lower storage cost; everything inside is transient. |
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
| Can a permanent database be cloned as transient? | Generally yes (`CREATE TRANSIENT DATABASE ... CLONE`), to cut dev storage cost; a transient object cannot be cloned as permanent. Confirm in the cloning docs. |
