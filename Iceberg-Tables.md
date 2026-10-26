## 1. What Are Iceberg Tables in Snowflake?

**Apache Iceberg** is an open table format: an abstraction layer over Parquet data files in object storage that adds ACID transactions, schema evolution, hidden partitioning, and table snapshots, and that many engines (Snowflake, Spark, Trino, Flink, etc.) can read and write.

**Snowflake Iceberg tables** combine Snowflake's query engine and platform features with **data that lives in cloud storage you (or Snowflake) manage in the open Iceberg format**. They are ideal for existing data lakes you cannot or choose not to move into Snowflake's native storage, and for multi-engine architectures that must avoid lock-in.

| Native Snowflake Table | Snowflake Iceberg Table |
| --- | --- |
| Data in Snowflake's proprietary micro-partition format | Data in **open Parquet files** plus Iceberg metadata |
| Storage fully inside Snowflake | Storage in **Snowflake-managed storage** *or* **your own S3 / GCS / Azure** location |
| Metadata only in Snowflake | Metadata in Snowflake **or** in an **external Iceberg catalog** |
| Only Snowflake reads and writes it | Other engines can read (and in some setups write) the same data |

Iceberg tables are available to all Snowflake accounts, on all cloud platforms and in all regions.

<br></br>

## 2. Iceberg Fundamentals (Needed to Understand the Snowflake Model)

```
        CATALOG  (maps table name -> current metadata file; atomic pointer swap)
           │
           ▼
   METADATA FILE  (schema, partition spec, snapshot list, properties)
           │
           ▼
   SNAPSHOT  (table state at a point in time)
           │
           ▼
   MANIFEST LIST ──► MANIFEST FILES  (list of data files + column stats)
                          │
                          ▼
                 PARQUET DATA FILES  (+ delete files / deletion vectors)
```

| Concept | Meaning |
| --- | --- |
| **Catalog** | Stores the **current metadata pointer** per table and supports **atomic updates** to it. Whoever owns the catalog controls who may commit |
| **Snapshot** | A consistent version of the table; every commit creates a new one. Enables **Time Travel** |
| **Manifest** | File listing data files with per-file stats (min/max, null counts), used for pruning |
| **Hidden partitioning** | Partition values are derived from **transforms** (e.g., `day(ts)`, `bucket(n, id)`) recorded in metadata; users filter on the source column, not a partition column |
| **Schema evolution** | Add, drop, rename, reorder columns without rewriting data (columns tracked by ID) |
| **Delete files / deletion vectors** | Row-level deletes without rewriting whole files: **position deletes** (v2), **deletion vectors** (v3) |
| **Spec versions** | Snowflake supports Iceberg **v1, v2, and v3** (excluding some features, such as row-level **equality deletes**) |

<br></br>

## 3. Building Blocks in Snowflake

| Object | Role | Notes |
| --- | --- | --- |
| **Iceberg table** | The Snowflake table object (`CREATE ICEBERG TABLE`) | Queried with normal SQL on a normal virtual warehouse |
| **Storage** | Where Parquet + metadata files live | **Snowflake storage** (`EXTERNAL_VOLUME = 'SNOWFLAKE_MANAGED'`) or **your cloud storage** through an external volume |
| **External volume** | Account-level object holding the IAM identity/credentials Snowflake uses to access **your** storage | One volume can serve many tables. Not needed with Snowflake storage |
| **Catalog** | Source of truth for the current metadata pointer | **Snowflake** itself, or an **external** Iceberg catalog |
| **Catalog integration** | Account-level object describing how to talk to an **external** catalog (Glue, Iceberg REST such as Open Catalog, object storage files, Delta) | Not needed when Snowflake is the catalog |
| **Catalog-linked database** | A Snowflake database **auto-synced** with a remote Iceberg REST catalog | Discovers namespaces and tables automatically |
| **Virtual warehouse** | Compute for queries, DML, and loads | Required, including for `CREATE ICEBERG TABLE` |

```
 Snowflake-managed (Snowflake is the catalog)         Externally managed (external catalog)
 ┌────────────────────────────┐                       ┌─────────────────────────────┐
 │ Snowflake: reads + writes  │                       │ Glue / Open Catalog / REST  │
 │ Snowflake owns catalog     │                       │ owns the metadata pointer   │
 └─────────────┬──────────────┘                       └──────────────┬──────────────┘
               │ Parquet + metadata                                   │ catalog integration
               ▼                                                      ▼
   Snowflake storage OR your bucket                    Your bucket (via external volume)
               ▲                                                      ▲
   other engines read via Open Catalog sync /          Spark / Trino / Flink etc. write;
   Horizon Iceberg REST API                            Snowflake reads (and can write via REST)
```

<br></br>

## 4. Table Types and Catalog Options

### 4.1 Overview

| Type | Catalog | Who Writes | Snowflake Platform Support | Typical Use |
| --- | --- | --- | --- | --- |
| **Snowflake-managed** | Snowflake | Snowflake (read and write). Other engines get read access through sync or Horizon APIs | **Full** | Lakehouse tables where Snowflake is the primary engine, but data must stay open |
| **Externally managed (Iceberg REST)** | Remote REST catalog (Snowflake Open Catalog / Polaris, Unity Catalog, others) | Other engines **and** Snowflake (write support for REST catalogs) | **Limited** | Shared lake where another engine/catalog is the authority |
| **Externally managed (AWS Glue)** | AWS Glue Data Catalog | Other engines. Snowflake reads through a Glue catalog integration, or reads and writes through Glue's Iceberg REST endpoint with a catalog-linked database | Limited | Existing AWS lake |
| **Iceberg files in object storage** | None (point at a metadata file path) | Other engines | Limited | Quick read of a known table version |
| **Delta Direct** | None (reads the Delta log) | Delta writers (e.g., Spark/Databricks) | Limited | Query Delta Lake tables from Snowflake without conversion |
| **Parquet Direct** (preview) | None | Anyone writing Parquet | Limited | Query raw Parquet in object storage faster than external tables |

### 4.2 Snowflake-Managed Iceberg Tables

* Snowflake **owns the catalog**, writes the data and metadata, and is the **single writer**.
* Full platform support: DML, transactions, clustering (for Snowflake-managed tables), standard streams, Time Travel, masking and row access policies, tags, sharing, replication of tables, and more.
* **Snowflake handles lifecycle maintenance**, including **compaction** (can be disabled per table), and expires snapshots and old metadata based on retention.
* Other engines read these tables through Open Catalog sync, the Horizon Iceberg REST Catalog API, or the Snowflake Catalog SDK (access rules and write limits are in section 11).
* Performance is close to a native table (some overhead for open-format storage).

### 4.3 Externally Managed Iceberg Tables

* An **external system owns the catalog and the table lifecycle**; Snowflake does **not** manage maintenance (compaction, snapshot expiry, orphan cleanup are done by your engine/tools).
* Snowflake finds the current state through the **catalog integration** and reads the Parquet files directly.
* Snowflake keeps its own view in sync by **refresh** (manual or automatic).
* **Write support** is available for tables managed by an **Iceberg REST catalog** (Iceberg v2 or v3): Snowflake commits to the remote catalog before updating its own table.
* Platform features are more limited (see section 15).

### 4.4 Catalogs You Can Use

| Catalog | Notes |
| --- | --- |
| **Snowflake** (Horizon) | Used for Snowflake-managed tables |
| **Snowflake Open Catalog** | Managed service for **Apache Polaris**, an open Iceberg REST catalog. Internal catalogs here can be written by third-party engines **and** Snowflake. You can also **sync Snowflake-managed tables to it** so other engines can query them |
| **Generic Iceberg REST catalogs** | E.g., Databricks Unity Catalog, self-hosted Polaris, others that follow the Iceberg REST spec. Can use **catalog-vended credentials** |
| **AWS Glue Data Catalog** | Supported through a Glue catalog integration, or through its Iceberg REST endpoint (which enables write support). Catalog-linked databases on Glue need lowercase, double-quoted identifiers |
| **Object storage metadata** | Point at a metadata file path |
| **Delta log in object storage** | Delta Direct |

### 4.5 Direct Tables: Delta Direct and Parquet Direct Notes

| Topic | Detail |
| --- | --- |
| **Delta Direct: supported** | Reads tables from engines using the latest Delta Lake version (4.0.0), including **deletion vectors** and **liquid clustering** (`minReaderVersion` 3). Partition schemes are read from the Delta log. Automated refresh is available |
| **Delta Direct: not supported** | Delta tables defined in the **AWS Glue Data Catalog**; Row Tracking, change data files, CDC, and protocol evolution; Parquet features such as field IDs, `INTERVAL`, `DECIMAL` above precision 38, one- or two-level `LIST` / `MAP`, unsigned integers, and `FLOAT16` |
| **Delta Direct: other** | `int96` works for `TIMESTAMP` but not `TIMESTAMP_NTZ`. Streams are **not** supported for Delta-based tables **with** partition columns (insert-only streams work without partition columns) |
| **Iceberg metadata for Delta tables** | With `ALLOW_WRITES = TRUE` on the external volume, Snowflake writes Iceberg metadata on a best-effort basis. With `FALSE` it writes none, but you can still query (use this for read-only access to a bucket another team owns) |
| **Parquet Direct** (preview) | Queries raw Parquet in object storage with better performance than external tables, with schema inference and evolution (`ENABLE_SCHEMA_EVOLUTION`) and Hive-style partition columns. Check its own limitations page before use |

<br></br>

## 5. Storage Options

| Aspect | Snowflake Storage (`SNOWFLAKE_MANAGED`) | Your Cloud Storage (external volume) |
| --- | --- | --- |
| Setup | None: no external volume, no IAM wiring | Create an external volume and grant Snowflake access |
| Who owns the files | Snowflake stores and manages them | You |
| Storage billing | **Snowflake** bills storage | **Your cloud provider** bills storage |
| Fail-safe | **Yes for permanent tables** (not for transient) | **No** (you must protect and recover data with your provider's tools) |
| Data protection | Snowflake Time Travel and Fail-safe | Snowflake Time Travel (for the retention window) plus your own backup/versioning |
| Best for | Fastest start, minimal ops, still open format | Data you must keep in your own buckets (compliance, shared lake) |

Docs call Snowflake storage the **recommended** way to create a Snowflake-managed Iceberg table. Cross-cloud and cross-region egress costs are covered in section 14.2.

### 5.1 Creating an External Volume (S3 Example)

```sql
CREATE EXTERNAL VOLUME iceberg_vol
  STORAGE_LOCATIONS = (
    (
      NAME                 = 's3-us-west-2'
      STORAGE_PROVIDER     = 'S3'
      STORAGE_BASE_URL     = 's3://my-lake-bucket/iceberg/'
      STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-iceberg-role'
    )
  )
  ALLOW_WRITES = TRUE;

DESC EXTERNAL VOLUME iceberg_vol;
-- Take STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID from the output
-- and add them to the IAM role's trust policy.
```

| Point | Detail |
| --- | --- |
| **Trust relationship** | Configure a **separate trust relationship per external volume** (IAM role trust, Azure service principal, or GCS service account) |
| **`ALLOW_WRITES`** | `TRUE` (default) for Snowflake-managed tables. `FALSE` gives a **read-only** volume, usable for externally managed tables. You can set `FALSE` only if no Snowflake-managed tables use that volume |
| **One or several volumes** | One volume can hold everything in subdirectories. Create several to secure locations differently (e.g., a **read-only volume** for externally managed tables, and a **read-write volume** for Snowflake-managed tables) |
| **Access** | You cannot access the storage locations in an external volume through a storage integration |
| **Encryption** | If using S3 SSE-KMS and external engines through Horizon, Snowflake must know the full KMS key ARN (set `KMS_KEY_ID` or allow `kms:DescribeKey`) |
| **Path stability** | After a Snowflake-managed table is created, its storage path does **not** change even if you rename the table |

### 5.2 File Layout

For Snowflake-managed tables without an explicit base location, Snowflake builds a path like `STORAGE_BASE_URL/database/schema/table_name.randomId/data/` and `/metadata/`. You can set `BASE_LOCATION` when creating a table in your own storage.

<br></br>

## 6. Catalog Integrations

### 6.1 Examples

```sql
-- AWS Glue
CREATE CATALOG INTEGRATION glue_int
  CATALOG_SOURCE    = GLUE
  CATALOG_NAMESPACE = 'my_glue_database'
  TABLE_FORMAT      = ICEBERG
  GLUE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-glue-role'
  GLUE_CATALOG_ID   = '123456789012'
  GLUE_REGION       = 'us-west-2'
  ENABLED           = TRUE;

-- Iceberg metadata files in object storage (no catalog service)
CREATE CATALOG INTEGRATION iceberg_files_int
  CATALOG_SOURCE = OBJECT_STORE
  TABLE_FORMAT   = ICEBERG
  ENABLED        = TRUE;

-- Delta Lake files in object storage (Delta Direct)
CREATE CATALOG INTEGRATION delta_int
  CATALOG_SOURCE = OBJECT_STORE
  TABLE_FORMAT   = DELTA
  ENABLED        = TRUE;

-- Iceberg REST catalog (Open Catalog / Polaris / Unity): syntax is catalog specific
-- (CATALOG_SOURCE = ICEBERG_REST with REST_CONFIG and REST_AUTHENTICATION blocks).
-- See "Configure a catalog integration" in the Snowflake docs for your catalog.
```

### 6.2 Notes

* A single catalog integration can serve **many tables** from the same external catalog.
* The Glue example above uses the classic Glue integration. Glue can also be reached as an **Iceberg REST catalog**, which is the route to **writes** and to a **catalog-linked database** (section 7.4).
* **Catalog-vended credentials:** with some REST catalogs, the catalog hands Snowflake **short-lived storage credentials**, so you don't have to wire a separate trust relationship to the bucket for each table.
* **Automated refresh** is configured on the catalog integration (`REFRESH_INTERVAL_SECONDS`) and enabled per table (`AUTO_REFRESH = TRUE`). For object-storage catalog integrations, automated refresh is supported only for `TABLE_FORMAT = DELTA`.
* **Shorter polling intervals** on frequently updated tables can degrade performance.

<br></br>

## 7. Creating Iceberg Tables

You need a **running warehouse** in the session, even for `CREATE ICEBERG TABLE`.

### 7.1 Snowflake-Managed on Snowflake Storage (Simplest)

```sql
CREATE ICEBERG TABLE orders (
  order_id   NUMBER,
  customer_id NUMBER,
  order_ts   TIMESTAMP_NTZ(6),
  amount     DECIMAL(12,2)
)
  CATALOG         = 'SNOWFLAKE'
  EXTERNAL_VOLUME = 'SNOWFLAKE_MANAGED';
```

### 7.2 Snowflake-Managed in Your Own Storage, Partitioned

```sql
CREATE OR REPLACE ICEBERG TABLE sales_events (
  event_id  NUMBER,
  event_ts  TIMESTAMP_NTZ(6),
  region    STRING,
  amount    DECIMAL(12,2)
)
  PARTITION BY (DAY(event_ts), region)
  CATALOG         = 'SNOWFLAKE'
  EXTERNAL_VOLUME = 'iceberg_vol'
  BASE_LOCATION   = 'sales_events/';
```

* Use **`CREATE ICEBERG TABLE ... AS SELECT`** and **`... LIKE`** for variants.
* **`TARGET_FILE_SIZE`** sets Parquet target file size (e.g., `AUTO` or fixed sizes).
* **`STORAGE_SERIALIZATION_POLICY`** (`COMPATIBLE` or `OPTIMIZED`) controls encoding and compression of data files. A poor choice can make tables **incompatible with external engines** or slow in Snowflake, so choose deliberately.
* **`ICEBERG_DEFAULT_DDL_COLLATION`** sets default collation for new string columns (Snowflake-managed tables only).
* **`ICEBERG_VERSION_DEFAULT`** sets the default Iceberg spec version for new tables (it can be altered on a catalog-linked database). Snowflake also documents a setting that makes Iceberg the **default metadata format** for new tables; see "Configure Iceberg as the default metadata format for tables" in the docs.

### 7.3 Externally Managed Tables

```sql
-- Iceberg REST catalog (reads and writes supported)
CREATE ICEBERG TABLE ext_orders
  EXTERNAL_VOLUME    = 'iceberg_vol'
  CATALOG            = 'my_rest_catalog_integration'
  CATALOG_TABLE_NAME = 'orders'
  AUTO_REFRESH       = TRUE;

-- AWS Glue
CREATE ICEBERG TABLE glue_orders
  EXTERNAL_VOLUME    = 'iceberg_vol'
  CATALOG            = 'glue_int'
  CATALOG_TABLE_NAME = 'orders';

-- Iceberg metadata file in object storage
CREATE ICEBERG TABLE file_orders
  EXTERNAL_VOLUME    = 'iceberg_vol'
  CATALOG            = 'iceberg_files_int'
  METADATA_FILE_PATH = 'orders/metadata/v12.metadata.json';

-- Delta Direct
CREATE ICEBERG TABLE delta_orders
  CATALOG         = 'delta_int'
  EXTERNAL_VOLUME = 'iceberg_vol'
  BASE_LOCATION   = 'delta/orders/'
  AUTO_REFRESH    = TRUE;
```

On creation, Snowflake performs an **initial metadata refresh**. For tables in a **standard** (non-linked) database that you intend to **write** to, the table must **already exist in the remote catalog** first.

### 7.4 Catalog-Linked Database

```sql
-- Syntax shown for orientation; confirm clause names in "CREATE DATABASE (catalog-linked)"
CREATE DATABASE lake_db
  LINKED_CATALOG = (
    CATALOG = 'my_rest_catalog_integration'
  );
```

A **catalog-linked database** is connected to an external **Iceberg REST catalog**. Snowflake automatically:

* **Discovers namespaces and tables** and registers them (namespaces appear as schemas).
* **Stays in sync** (automated refresh is on by default) and supports **creating/dropping schemas and Iceberg tables**, which are created **in the remote catalog too**.
* Lets you **read and write** remote tables, with **no need to create each table manually**.

| Consideration | Detail |
| --- | --- |
| Filtering | `ALLOWED_NAMESPACES` / `BLOCKED_NAMESPACES` limit auto-discovery |
| Nested namespaces | Flattened into schema names using `NAMESPACE_FLATTEN_DELIMITER` (with `NAMESPACE_MODE`) |
| Access control | Snowflake **does not sync** remote catalog users/roles; manage Snowflake privileges separately |
| Identifier case | Glue and other lowercase-only REST catalogs require **lowercase, double-quoted** identifiers |
| Drops and renames | Dropping or renaming a table remotely causes Snowflake to drop it and (for renames) recreate it under the new name; `UNDROP ICEBERG TABLE` is not supported here |
| Objects allowed | Schemas, externally managed Iceberg tables, and database roles |
| Defaults | `ICEBERG_VERSION_DEFAULT`, `ICEBERG_MERGE_ON_READ_BEHAVIOR` can be altered on the database |
| Governance | Masking policies and tags supported; replication and cloning of the linked database are not |

### 7.5 Parquet and Metadata Requirements

| Requirement | Detail |
| --- | --- |
| **File format** | Parquet only; Parquet files that use the **unsigned integer** logical type are not supported |
| **`LIST` columns** | The three-level structure with the `element` keyword is supported; files using the obsolete `array` keyword layout must be regenerated |
| **Manifests** | No duplicate files within the **same snapshot**, or Snowflake returns an error naming the duplicate path |
| **Parquet metadata** | Must be valid UTF-8, and statistics (such as row counts and null counts) must match the data. Snowflake surfaces corruption as errors, but inconsistent tables may still be creatable or queryable |
| **Metadata files** | Metadata files don't identify the latest snapshot, and you cannot change data or snapshot locations with `ALTER`; recreate the table with `CREATE OR REPLACE` instead |

<br></br>

## 8. Loading and Modifying Data

| Operation | Snowflake-Managed | Externally Managed |
| --- | --- | --- |
| `SELECT` | Yes | Yes |
| `INSERT`, `UPDATE`, `DELETE`, `MERGE` | Yes | Yes for REST catalogs (v2 / v3), autocommit only |
| `CREATE ... AS SELECT` | Yes | Yes (limited for some catalogs, e.g., Glue and Unity: create empty table then `INSERT ... SELECT`, which is not atomic as one transaction) |
| `COPY INTO <table>` | Yes (limited support: see usage notes) | Limited |
| Snowpipe | Limited (public preview, per the docs' partitioning support matrix) | Limited |
| Snowpipe Streaming | Not supported (per the partitioning matrix; verify for unpartitioned tables) | Not supported (same caveat) |

Transactions, streams, and other feature differences are summarized in section 15.

**Loading Parquet with `COPY INTO` / Snowpipe:** the recommended file format options are `BINARY_AS_TEXT = FALSE`, `USE_LOGICAL_TYPE = TRUE`, `USE_VECTORIZED_SCANNER = TRUE`, and `REPLACE_INVALID_CHARACTERS = TRUE`.

### 8.1 Row-Level Deletes: Copy-on-Write vs Merge-on-Read

* **Copy-on-write:** rewrite whole data files that contain changed rows (more write cost, simpler reads).
* **Merge-on-read:** write small **delete files** (position deletes in v2, **deletion vectors** in v3) and reconcile at read time (cheaper writes, reads need cleanup via compaction).
* Snowflake supports **position deletes** (v2) and **deletion vectors** (v3); **equality deletes are not supported**.
* Behaviour is controlled by `ICEBERG_MERGE_ON_READ_BEHAVIOR` (set to `'DISABLED'` for copy-on-write) at table, schema, or database level.
* **Dangling or excessive position deletes** can block table creation and refresh, so run regular compaction and maintenance in your external engine (for example, Spark `rewrite_data_files` with delete-file options).

### 8.2 Partitioning

| Style | How | When |
| --- | --- | --- |
| **Hidden partitioning** (default) | `PARTITION BY (transform(col), ...)`, `PATH_LAYOUT = FLAT` (default); files in a flat directory | Best default; partition info lives in metadata |
| **Hierarchical (Hive-style) paths** | `PATH_LAYOUT = HIERARCHICAL` with `PARTITION BY`; files under paths like `country=US/year=2026/` | Interoperability with engines expecting Hive-style paths |

* Common transforms: `YEAR`, `MONTH`, `DAY`, `HOUR`, `TRUNCATE`, `BUCKET`, identity.
* **Iceberg `DAY()` / `MONTH()` / `YEAR()` transforms differ from Snowflake SQL functions**: the transform makes one partition per calendar day/month/year, while the SQL function extracts a part. Two months of data with `DAY()` gives about 61 partitions.
* The `BUCKET` transform can hurt performance for queries with conditional filters.
* **Partition evolution** is supported on Snowflake-managed tables; on externally managed tables only when done with an external engine.
* The combined size of partition transform outputs for one row cannot exceed **1024 bytes**.
* Snowflake always writes hierarchical paths under `/data`, and may serialize values differently from other engines (for example, `~` encoding).

<br></br>

## 9. Metadata, Snapshots, Refresh, and Time Travel

### 9.1 Snowflake-Managed Tables

* Snowflake **generates Iceberg metadata** (v2 or v3 spec) **periodically**, writing it to the table's location. Each new metadata file captures all DML/DDL since the previous one.
* Generate metadata **on demand**: `SELECT SYSTEM$GET_ICEBERG_TABLE_INFORMATION('db.schema.table');`
* Review metadata-generation attempts in **query history** (user `SYSTEM`, filter for that function).
* **Retention and cleanup:** governed by `DATA_RETENTION_TIME_IN_DAYS` (account, database, schema, or table level). Snowflake deletes old metadata, manifest lists, and manifests for **expired snapshots** and **dropped tables**, usually **7 to 14 days after expiry** (asynchronous), and never deletes the files for the **current** table state.
* `UNDROP ICEBERG TABLE` works within the retention period.
* `history.expire.min-snapshots-to-keep` is **not supported** for any Iceberg table type.
* For Snowflake-managed tables, DDL statements are processed individually, so the metadata Snowflake produces can differ from what other catalogs generate.

### 9.2 Externally Managed Tables

* Snowflake **never deletes** your Iceberg metadata or snapshots; retention in your catalog and storage is **your** responsibility.
* Snowflake sets the table's `DATA_RETENTION_TIME_IN_DAYS` automatically to the **smaller of**:
  * the `history.expire.max-snapshot-age-ms` value in the current metadata file (converted to days, rounded down), and
  * **1 day (Standard)** or **5 days (Enterprise and above)**.
* You **cannot change it in Snowflake**; change `history.expire.max-snapshot-age-ms` in the metadata and refresh the table.
* **Time Travel** to snapshots created after table creation works only if you **refresh before the snapshot expires**.

### 9.3 Refreshing Externally Managed Tables

```sql
ALTER ICEBERG TABLE ext_orders REFRESH;                                   -- latest snapshot
ALTER ICEBERG TABLE file_orders REFRESH 'orders/metadata/v13.metadata.json';  -- specific metadata file
```

* **Automated refresh** (`AUTO_REFRESH = TRUE`, configured as described in section 6.2) synchronizes schema changes together with DML. DDL-only changes need a **manual refresh**.
* **Refresh often**, and **align refreshes with maintenance** (snapshot expiry, compaction) by refreshing after each maintenance run. Refresh time depends on the number of **snapshots** and **data files**.
* Monitoring:

```sql
SELECT * FROM TABLE(INFORMATION_SCHEMA.ICEBERG_TABLE_SNAPSHOT_REFRESH_HISTORY(
       TABLE_NAME => 'EXT_ORDERS'));
SELECT * FROM TABLE(INFORMATION_SCHEMA.ICEBERG_TABLE_FILES(TABLE_NAME => 'EXT_ORDERS'));
```

*Check the exact signatures of these two table functions in the SQL reference.*

### 9.4 Time Travel Queries

```sql
SELECT * FROM orders AT (TIMESTAMP => '2026-10-01 00:00:00'::TIMESTAMP_LTZ);
SELECT * FROM orders BEFORE (STATEMENT => '<query_id>');
```

For Snowflake-managed tables, third-party engines can also time travel after **syncing to Open Catalog** or using the **Catalog SDK**.

<br></br>

## 10. Table Maintenance

| Task | Snowflake-Managed | Externally Managed |
| --- | --- | --- |
| **Compaction** (merge small files, apply deletes) | **Automatic** by Snowflake; can be disabled per table (`ENABLE_DATA_COMPACTION = FALSE`) | **You** run it in your engine (e.g., Spark `rewrite_data_files`) |
| **Snapshot expiry** | Automatic via data retention | You (e.g., Spark `expire_snapshots`) |
| **Metadata cleanup** | Automatic | You |
| **Orphan file removal** | **Not supported** by Snowflake; failed DML can leave untracked Parquet files, so contact Snowflake Support if storage doesn't match | You (e.g., `remove_orphan_files`) |
| **Target file size** | `TARGET_FILE_SIZE` | Engine writer settings (and Snowflake's setting for its own writes) |
| **Statistics quality** | Written by Snowflake | Your writer must write **min/max, NDV, null counts** |

**Why writer statistics matter for externally managed tables:** missing min/max values hurt pruning, and missing **number of distinct values (NDV)** can cause the optimizer to pick a bad join order, leading to **join explosion**. Configure Spark/Trino writers to emit full Parquet statistics and keep them updated.

<br></br>

## 11. Interoperability with Other Engines

| Mechanism | What It Does |
| --- | --- |
| **Snowflake Open Catalog sync** | Sync a Snowflake-managed table to Open Catalog (Polaris) so Spark, Trino, Flink, etc. can query it. Externally synced catalogs are **read-only** for those engines |
| **Horizon Iceberg REST Catalog API** | External query engines access **Snowflake-managed** Iceberg tables via Snowflake's REST catalog, with **vended credentials**, and fine-grained policies (masking, row access) can be enforced. Reading v2 and v3 is supported. Writes are supported with restrictions |
| **Snowflake Catalog SDK** | Lets Iceberg libraries (for example, Spark) read Snowflake-managed tables through the SDK |
| **Open Catalog as the catalog** | Snowflake and other engines both read and write tables managed by Open Catalog |
| **Direct object storage access** | Engines read Parquet and metadata straight from your bucket (for tables in your own storage) |

**Rules to remember**

* For **Snowflake-managed** tables, **third-party clients cannot directly append, delete, or upsert**; writes go through Snowflake (or via Horizon REST with its restrictions).
* Through Horizon, external engines can access **only Snowflake-managed** Iceberg tables (not native tables, remote tables, or externally managed/Delta-based tables).
* Limits on external writes via Horizon include: no base location in `CREATE TABLE`, no CTAS from external engines, no equality deletes, no tags/branches, no writing to dynamic tables, shared tables, or registering tables, and no snapshot rollback or v2 to v3 upgrade.
* **Cloned or converted tables** have extra credential-vending restrictions.

<br></br>

## 12. Converting Between Types

```sql
-- Convert an externally managed table to Snowflake-managed (Snowflake becomes the catalog)
ALTER ICEBERG TABLE ext_orders CONVERT TO MANAGED
  BASE_LOCATION = 'converted/orders/';
```

* Gives **full Snowflake platform support** and Snowflake-managed maintenance.
* Conversion **moves catalog ownership to Snowflake**, so plan for external engines to stop writing through the old catalog afterwards (guidance, not an enforced rule). I found no documented conversion in the opposite direction.
* **Limitation:** tables that use **externally managed write support** (Snowflake writing to a remote REST catalog) **cannot** be converted to Snowflake-managed.
* For converted tables, Snowflake only deletes metadata **generated after conversion**, and clustering applies only to files created after conversion or later modified by DML.
* Writing to a table converted from externally managed to Snowflake-managed through an external engine is not supported.

<br></br>

## 13. Security, Governance, and Sharing

| Area | Detail |
| --- | --- |
| **Privileges** | `CREATE ICEBERG TABLE` on the schema; `USAGE` on the **external volume**; `USAGE` on the **catalog integration**; creating those objects needs `CREATE EXTERNAL VOLUME` and `CREATE INTEGRATION` (account-level) |
| **Standard RBAC** | `SELECT`, `INSERT`, etc. on the table as with native tables |
| **Policies and tags** | Masking policies, row access policies, and tags are supported (including for externally managed tables via `ALTER ICEBERG TABLE`) |
| **External engine access** | Fine-grained policies can be enforced for Horizon access |
| **Storage security** | IAM role per external volume; least-privilege (read-only volume for externally managed tables); optional customer KMS keys |
| **Sharing** | Snowflake-managed tables can be shared; Native App sharing is read-only for consumers; externally managed table sharing has restrictions that differ between catalog-linked and standard setups |
| **Unsupported on Iceberg tables** | Hybrid tables, Snowflake encryption, and temporary tables (as listed in the docs) |

```sql
GRANT USAGE ON EXTERNAL VOLUME iceberg_vol TO ROLE data_engineer;
GRANT USAGE ON INTEGRATION glue_int        TO ROLE data_engineer;
GRANT CREATE ICEBERG TABLE ON SCHEMA lake.raw TO ROLE data_engineer;
```

<br></br>

## 14. Performance and Cost

### 14.1 Performance

| Factor | Guidance |
| --- | --- |
| **Snowflake-managed vs native** | Close to native performance; some overhead from open-format storage |
| **Externally managed** | Depends on **statistics quality**, file sizes, and delete-file buildup |
| **File size** | Avoid many small files; use `TARGET_FILE_SIZE` and compaction |
| **Partitioning** | Use time-based transforms for time-filtered queries; avoid over-partitioning; `BUCKET` can hurt conditional filters |
| **Clustering** | Available only for Snowflake-managed tables; see section 15 for details and the partitioned-table caveat |
| **Refresh frequency** | Refresh externally managed tables regularly to keep refresh times short |
| **Warehouse size** | A **larger warehouse speeds up table creation** when Snowflake must scan many files for statistics, but **does not speed up query runtime** by parallelizing column scans |
| **Serialization policy** | Wrong `STORAGE_SERIALIZATION_POLICY` can hurt compatibility or speed |
| **Cache** | Local warehouse cache applies as with native tables, while result and metadata caches behave as usual |

### 14.2 Billing Summary

| Cost Item | Who Charges |
| --- | --- |
| Warehouse compute, cloud services | Snowflake |
| Storage with **customer-managed external volume** | **Your cloud provider** (Snowflake does not bill storage) |
| Storage with **Snowflake storage** (`SNOWFLAKE_MANAGED`) | **Snowflake** |
| Automated refresh; external engine access through Horizon | Snowflake (separate usage) |
| Cross-cloud / cross-region transfer | Snowflake-managed tables: Snowflake bills egress under `DATA_LAKE` in `DATA_TRANSFER_HISTORY` (for Snowflake storage, all egress; for your storage, Snowflake writes cross-region). Reads and writes by external engines to your storage: your cloud provider bills egress |
| Metadata retained after drop/expiry | Your cloud provider, for the period before deletion completes |

```sql
-- Storage metrics (active bytes are shown for tracking, but not billed by Snowflake for external storage)
SELECT table_catalog, table_schema, table_name, active_bytes
FROM snowflake.account_usage.table_storage_metrics
WHERE table_name = 'SALES_EVENTS';
```

<br></br>

## 15. Feature Support Summary

| Feature | Snowflake-Managed | Externally Managed |
| --- | --- | --- |
| Read | Yes | Yes |
| Write (DML) | Yes | Yes with Iceberg REST catalogs (autocommit only) |
| Multi-statement transactions | Yes | No for writes |
| Time Travel | Yes (retention-based) | Yes for snapshots you refresh in time |
| Clustering | Yes (set with `CREATE` / `ALTER ICEBERG TABLE`). The docs' partitioning matrix lists clustering as unsupported for partitioned tables, so test that combination | No |
| Standard / append-only streams | Yes | No (**insert-only** streams only) |
| Cloning | Yes | Docs are inconsistent: the general list says unsupported, the partitioning matrix shows support with usage notes. Verify |
| Replication | Yes (Snowflake-managed tables) | No |
| Snowflake schema evolution | Yes (Iceberg schema evolution) | No (use the external engine) |
| Masking / row access / tags | Yes | Yes |
| Fail-safe | Only with Snowflake storage (permanent tables) | No |
| Automatic compaction and cleanup | Yes | No (you maintain) |
| Collation | Yes | No |
| Sharing | Yes | Restricted |

*This area changes quickly. Confirm each row against the current "Apache Iceberg tables" considerations page before relying on it.*

<br></br>

## 16. Iceberg Tables vs Native Tables vs External Tables

| Aspect | Native Table | Iceberg (Snowflake-managed) | Iceberg (Externally managed) | External Table |
| --- | --- | --- | --- | --- |
| Data format | Snowflake micro-partitions | Open Parquet + Iceberg metadata | Open Parquet + Iceberg metadata | Files in object storage (CSV, Parquet, etc.) |
| Storage owner | Snowflake | Snowflake or you | You | You |
| Writes from Snowflake | Yes | Yes | Yes (REST catalogs) | **No** (read-only) |
| Multi-engine | No | Reads via Open Catalog/Horizon | Native (catalog shared) | Read only through Snowflake |
| ACID / schema evolution | Snowflake | Iceberg | Iceberg | No table-format guarantees |
| Pruning | Micro-partition metadata | Iceberg manifests + stats | Iceberg manifests + stats | File-level partition columns |
| Maintenance | Automatic | Automatic compaction and cleanup | You | n/a |
| Performance | Best | Near native | Depends on stats and maintenance | Slowest |
| Time Travel / Fail-safe | Full | Time Travel; Fail-safe only on Snowflake storage | Limited; no Fail-safe | None |

<br></br>

## 17. Choosing an Approach

```
 Do other engines need to WRITE to the same tables?
   │ No ──► Is storage location mandated (your own bucket)?
   │           │ No  ──► Snowflake-managed table on SNOWFLAKE_MANAGED storage (simplest, Fail-safe)
   │           │ Yes ──► Snowflake-managed table on your external volume
   │           │         (sync to Open Catalog / Horizon so other engines can READ)
   │ Yes ─► Which catalog is the authority?
   │           │ Open Catalog / Unity / other REST ──► externally managed via catalog-linked database
   │           │ AWS Glue                           ──► externally managed (Glue integration)
   │           │ None, Delta Lake tables            ──► Delta Direct
   ▼
 Need full Snowflake features later? ──► convert externally managed table to Snowflake-managed
```

| Scenario | Recommended Approach |
| --- | --- |
| New lakehouse, Snowflake is main engine, open format desired | Snowflake-managed on Snowflake storage |
| Compliance requires data in your own bucket | Snowflake-managed on external volume |
| Spark/Databricks write, Snowflake reports | Externally managed via REST catalog (catalog-linked database) |
| Existing AWS Glue lake | Externally managed with Glue integration, refresh frequently |
| Delta Lake tables that Databricks owns | Delta Direct |
| Replace slow external tables on raw Parquet | Parquet Direct (preview) |
| Need Snowflake streams, clustering, replication on lake data | Convert to Snowflake-managed |

<br></br>

## 18. Monitoring and Troubleshooting

```sql
-- Metadata generation attempts for Snowflake-managed tables (user SYSTEM)
SELECT query_id, start_time, execution_status, error_message
FROM snowflake.account_usage.query_history
WHERE user_name = 'SYSTEM'
  AND query_text ILIKE '%SYSTEM$GET_ICEBERG_TABLE_INFORMATION%'
  AND start_time >= DATEADD(day, -3, CURRENT_TIMESTAMP())
ORDER BY start_time DESC;

-- Refresh history for an externally managed table
SELECT * FROM TABLE(INFORMATION_SCHEMA.ICEBERG_TABLE_SNAPSHOT_REFRESH_HISTORY(
       TABLE_NAME => 'EXT_ORDERS'));

-- Cross-region data lake transfer costs
SELECT *
FROM snowflake.account_usage.data_transfer_history
WHERE transfer_type = 'DATA_LAKE'
  AND start_time >= DATEADD(day, -30, CURRENT_TIMESTAMP());
```

| Symptom | Likely Cause | Action |
| --- | --- | --- |
| Table shows old data | Not refreshed | `ALTER ICEBERG TABLE ... REFRESH` or enable auto-refresh |
| Refresh or create is slow | Many snapshots/files, missing statistics, dangling position deletes | Compact and expire snapshots in the external engine; use a larger warehouse for creation |
| Join explosion on external table | Missing NDV statistics | Fix writer statistics |
| Time Travel not available | Snapshot expired before refresh | Refresh more often; raise `history.expire.max-snapshot-age-ms` |
| Access denied reading data | IAM trust, KMS key, or volume `ALLOW_WRITES` mismatch | Recheck external volume trust and key permissions |
| Storage bigger than Snowflake shows | Orphan files from failed DML | Contact Snowflake Support |
| External engine can't read a Snowflake-managed table | Wrong serialization policy or access path | Check `STORAGE_SERIALIZATION_POLICY`, Open Catalog sync, or Horizon setup |

<br></br>

## 19. Best Practices and Pitfalls

**Do**

* Prefer **Snowflake storage** unless you must keep files in your own bucket.
* Use **Snowflake-managed tables** when Snowflake is the primary writer; use **catalog-linked databases** for REST catalogs.
* Use **separate external volumes** (read-only vs read-write) to apply least privilege.
* **Refresh frequently** and right after maintenance for externally managed tables; keep snapshot counts and file counts under control.
* Ensure **complete Parquet statistics** from every writer.
* Use the recommended **Parquet file format options** when loading with `COPY INTO` or Snowpipe.
* Set **retention** deliberately (and remember cleanup is asynchronous).
* Plan **egress** when storage and Snowflake are in different regions or clouds.

**Don't**

* Don't let **two engines write** to a Snowflake-managed table's files directly.
* Don't expect Snowflake to **maintain** externally managed tables (compaction, snapshot expiry, orphan files), or assume **Fail-safe** protects data in your own storage (section 5).
* Don't scale the warehouse up to speed up queries on an external table with poor statistics; fix the statistics.
* Don't assume SQL behaviour carries over to Iceberg partition transforms (section 8.2), or rely on features unsupported for your table type, such as equality deletes or multi-statement transactions (sections 8.1 and 15).

<br></br>

## 20. Quick Interview Q&A

| Question | Short Answer |
| --- | --- |
| What is an Iceberg table in Snowflake? | A table using the open Apache Iceberg format (Parquet data + Iceberg metadata) in cloud storage, queried and managed with Snowflake's engine and features. |
| Snowflake-managed vs externally managed? | Snowflake-managed: Snowflake is the catalog and single writer, full platform support, automatic maintenance. Externally managed: an external catalog owns metadata and lifecycle, Snowflake reads (and writes with REST catalogs) with limited features. |
| What is an external volume? | An account-level object holding the credentials/identity Snowflake uses to access your cloud storage for Iceberg tables. |
| What is a catalog integration? | An account-level object describing how Snowflake connects to an external Iceberg catalog (Glue, REST, object storage, Delta). |
| What storage options exist? | Snowflake storage (`SNOWFLAKE_MANAGED`, Snowflake bills storage, Fail-safe for permanent tables) or your own bucket via an external volume (provider bills storage, no Fail-safe). |
| What is a catalog-linked database? | A Snowflake database auto-synced with an Iceberg REST catalog that discovers namespaces and tables and supports reads, writes, and creating tables. |
| What is Snowflake Open Catalog? | A managed service for Apache Polaris, an open Iceberg REST catalog used by Snowflake and other engines. |
| How do other engines read a Snowflake-managed table? | Via Open Catalog sync, the Horizon Iceberg REST Catalog API, or the Catalog SDK. They cannot directly write into it. |
| How is Time Travel handled for external tables? | Retention is derived from the catalog's snapshot-age setting (capped at 1 or 5 days by edition) and you must refresh before snapshots expire. |
| Who compacts files? | Snowflake for Snowflake-managed tables (can be disabled); you for externally managed tables. |
| Row-level deletes support? | Position deletes (v2) and deletion vectors (v3); equality deletes unsupported. |
| What are Delta Direct and Parquet Direct? | Iceberg tables created directly over Delta Lake files or raw Parquet files in object storage, without a separate Iceberg catalog. |
| Can you cluster Iceberg tables? | Snowflake-managed tables yes; externally managed no. |
| What are the main cost differences? | Customer storage billed by your cloud provider; Snowflake bills compute, cloud services, refresh, Snowflake storage, and some cross-region egress. |
| How do you move to full Snowflake features? | `ALTER ICEBERG TABLE ... CONVERT TO MANAGED`. |
| What causes join explosion on external tables? | Missing NDV statistics in Parquet/manifest files. |
| Why are Iceberg DAY() and SQL DAY() different? | The partition transform creates a partition per calendar day, while the SQL function extracts the day of month. |
| What can go wrong with Fail-safe? | It only applies to permanent tables on Snowflake storage, so tables on your own storage need your own protection. |
