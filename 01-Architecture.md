# Snowflake Architecture
<br></br>
## 1. High-Level Architecture Overview

Snowflake utilizes a **Multi-Cluster Shared-Data Architecture**. It separates the database into three independently scalable layers: **Centralized Storage**, **Compute (Virtual Warehouses)**, and **Cloud Services**.

It is best described as a **hybrid of shared-disk and shared-nothing**: like shared-disk, all compute sees one central copy of data; like shared-nothing, queries run on MPP clusters with local (cached) data and no shared CPU/memory.

```
+---------------------------------------------------------------------------------+
|                              CLOUD SERVICES LAYER                               |
|   Authentication  |  Infrastructure Mgmt  |  Query Parsing  |  Access Control   |
|   Metadata Mgmt   |  Query Optimization   |  Security       |  Transaction Control |
+---------------------------------------------------------------------------------+
                                       |
                +----------------------+----------------------+
                |                                             |
+-------------------------------+             +-------------------------------+
|    VIRTUAL WAREHOUSE A        |             |    VIRTUAL WAREHOUSE B        |
|  (Compute Engine: ETL Work)   |             |  (Compute Engine: BI / Apps)  |
|  +-----+ +-----+ +-----+      |             |  +-----+ +-----+ +-----+      |
|  | Node| | Node| | Node|      |             |  | Node| | Node| | Node|      |
|  +-----+ +-----+ +-----+      |             |  +-----+ +-----+ +-----+      |
+-------------------------------+             +-------------------------------+
                |                                             |
                +----------------------+----------------------+
                                       |
+---------------------------------------------------------------------------------+
|                           CENTRALIZED STORAGE LAYER                             |
|          Amazon S3   |   Microsoft Azure Blob Storage   |   Google Cloud        |
|               (Compressed, Hybrid Columnar, Micro-Partitions)                   |
+---------------------------------------------------------------------------------+

```

<br></br>

## 2. The 3 Architectural Layers

### Layer 1: Centralized Storage (Data Persistence)

* **Underlying Engine:** Cloud object storage (Amazon S3, Azure Blob Storage, or Google Cloud Storage).
* **Storage Format:** Data is automatically converted into an optimized, **proprietary hybrid columnar format** (PAX-style: rows grouped into micro-partitions, stored column-by-column inside each).
* **Storage is fully managed:** Users cannot see or access the underlying files directly; data is only reachable through SQL via a virtual warehouse.
* **Micro-Partitions:**
  * Physical data files containing between **50 MB to 500 MB** of uncompressed data (smaller once compressed).
  * Stored as **immutable objects**.
  * Columns within each micro-partition are compressed individually; Snowflake automatically picks the encoding per column (e.g., Run-Length, Dictionary).
  * Each micro-partition has a header with column offsets, so a query reads only the **columns** it needs from a partition (column pruning).
* **Key Mechanisms:**
  * **Immutability:** Updates and deletes do not overwrite existing files; instead, new micro-partition versions are created and the metadata pointers change. This enables native **Time Travel**, **Fail-safe**, and **Zero-Copy Cloning**.
  * **Automatic Partitioning:** Data partitioning occurs automatically during ingestion (in order of arrival) without explicit partition keys.
  * **Storage includes more than tables:** Query results, stage files, and Time Travel / Fail-safe historical versions are also held in this layer.

### Layer 2: Compute Layer (Virtual Warehouses)

* **Physical Realization:** MPP (Massively Parallel Processing) compute clusters composed of virtual machine instances provisioned from the cloud provider (e.g., AWS EC2).
* **Stateless Processing (no persistent table data):** Compute nodes do not persistently store table data. They fetch micro-partitions on-demand from the Centralized Storage layer into local high-speed SSD/memory (**Local Disk Caching**). The cache is lost when the warehouse is suspended.
* **Isolation Mechanics:**
  * Virtual Warehouses do not share CPU, memory, or local storage.
  * Multiple independent warehouses can read from and write to the same underlying storage layer concurrently, with **no compute contention between warehouses**.
  * Table-level write coordination still exists: concurrent `UPDATE` / `DELETE` / `MERGE` on the **same table** are serialized using table-level locks (queued, not failed). Reads are never blocked by writes.
* **Elasticity:**
  * **Auto-Suspend / Auto-Resume:** Warehouses suspend after a configurable idle period and resume automatically on the next query (typically in seconds). Billing is per-second with a **60-second minimum** on each resume.
* **Scaling Vectors:**
  * **Scale-Up (Vertical Elasticity):** Resizing a warehouse (e.g., `X-Small` to `6X-Large`) dynamically allocates more CPU/RAM per cluster to accelerate single, complex, resource-intensive queries. Resizing affects **new** queries; running queries are unaffected.
  * **Scale-Out (Horizontal Elasticity):** Multi-Cluster Warehouses (Enterprise edition and above) automatically add or remove identical compute clusters to scale **concurrency** (not single-query speed).
* **Resource pressure behaviour:** If a query's working set exceeds memory it **spills to local SSD**, then to **remote storage** (a classic sign that the warehouse should be sized up). If all slots are busy, queries **queue** until capacity frees up.

### Layer 3: Cloud Services (The Control Plane)

* **Engine:** A distributed, stateless service layer backed by a transactional metadata store (FoundationDB). Runs on Snowflake-managed compute shared across customers.
* **Key Responsibilities:**
  * **Metadata Management:** Tracks micro-partition file pointers, offsets, column min/max ranges, distinct value counts, and modification histories.
  * **Query Optimization & Compilation:** Parses SQL, applies rule/cost-based optimizations using metadata, performs partition pruning, and generates execution plans for Virtual Warehouses.
  * **Transaction Control:** Enforces ACID compliance across concurrent workloads using **MVCC / Snapshot-style isolation**. At the SQL level, Snowflake exposes the **READ COMMITTED** isolation level for transactions.
  * **Security & Governance:** Handles user authentication, RBAC (Role-Based Access Control), encryption key management, and session state.
  * **Infrastructure Management:** Provisions, resizes, and suspends warehouses; handles failures and upgrades without downtime.
* **Billing note:** Cloud services usage is only charged when it exceeds **10% of daily virtual warehouse credit usage**.

### Beyond the three layers: Serverless Compute

Some features run on **Snowflake-managed compute** instead of a user-sized warehouse, and are billed by actual usage: **Snowpipe**, **Automatic Clustering**, **Search Optimization Service**, **Materialized View maintenance**, and **serverless Tasks**.

<br></br>

## 3. Query Lifecycle (End-to-End Flow)

```
 Client (UI / Driver / API)
        |
        v
 [1] CLOUD SERVICES: authenticate, authorize (RBAC), check RESULT CACHE
        |        (hit -> return immediately, no warehouse needed)
        v
 [2] CLOUD SERVICES: parse -> optimize -> use metadata for PARTITION PRUNING -> plan
        |
        v
 [3] VIRTUAL WAREHOUSE: nodes execute plan in parallel
        |        reads only needed micro-partitions / columns
        |        (local SSD cache first, else remote object storage)
        v
 [4] Results returned to client; stored in RESULT CACHE
```

**Write path (DML / loads):** The warehouse writes **new micro-partition files** to object storage; Cloud Services then **atomically commits the metadata change** (new snapshot). Readers continue to see the previous snapshot until the commit, which is how read/write isolation works without blocking.

<br></br>

## 4. Micro-Partition Metadata & Pruning

* **Metadata kept per micro-partition:** min/max value of each column, number of distinct values, NULL counts, and other optimization properties.
* **Partition Pruning:** The optimizer compares query filters against min/max ranges and skips micro-partitions that cannot contain matches, so only a fraction of files is ever scanned.
* **Natural Clustering:** Data is clustered by load order (e.g., by date for append-only tables), so pruning is effective on those columns without any setup.
* **Clustering Keys / Automatic Clustering:** For very large tables where natural order degrades (overlapping min/max ranges), a clustering key can be defined and Snowflake re-organizes micro-partitions in the background (serverless). Quality is measured via **clustering depth / overlap** (`SYSTEM$CLUSTERING_INFORMATION`).
* **Metadata-only answers:** Queries such as `COUNT(*)`, `MIN()`, `MAX()` on a whole table can be answered from metadata without scanning data or using a warehouse.

<br></br>

## 5. Caching Architecture (Three Levels)

| Cache | Lives In | What it Holds | Lifetime / Key Behaviour |
| --- | --- | --- | --- |
| **Result Cache** | Cloud Services | Final results of previous queries | 24 hours; each reuse extends it (up to 31 days). Reused only if query text matches, underlying data is unchanged, no non-deterministic functions, and the role has access. Costs **no warehouse compute**. |
| **Metadata Cache** | Cloud Services | Table statistics (row counts, min/max, etc.) | Always maintained; serves metadata-only queries and pruning. |
| **Local Disk (Warehouse) Cache** | Warehouse SSD / memory | Raw micro-partition data (columns) read from storage | Per-warehouse; cleared on suspend; evicted by LRU. |

The **remote storage layer** is the source of truth beneath these caches.

<br></br>

## 6. Virtual Warehouse Sizing Reference

Each size up **doubles** the compute resources and the credit consumption rate.

| Size | Credits / Hour |
| --- | --- |
| X-Small | 1 |
| Small | 2 |
| Medium | 4 |
| Large | 8 |
| X-Large | 16 |
| 2X-Large | 32 |
| 3X-Large | 64 |
| 4X-Large | 128 |
| 5X-Large | 256 |
| 6X-Large | 512 |

**Multi-Cluster modes:**

* **Maximized:** `MIN_CLUSTER_COUNT = MAX_CLUSTER_COUNT` (> 1); all clusters run whenever the warehouse is running.
* **Auto-scale:** `MIN` < `MAX`; clusters start and stop with load. Scaling policy: **Standard** (favours avoiding queuing) or **Economy** (favours saving credits, tolerates some queuing).

**Rule of thumb:** *Scale up* for slow, complex queries (or spilling); *scale out* for many concurrent users/queries (queuing).

<br></br>

## 7. Data Protection in the Storage Layer

| Feature | Purpose | Details |
| --- | --- | --- |
| **Time Travel** | Query, clone, or restore data as it existed in the past; undo changes and drops | 1 day by default; up to **90 days** on Enterprise edition and above (permanent tables). Transient and temporary tables: max 1 day. |
| **Fail-safe** | Last-resort recovery by Snowflake after Time Travel ends | **7 days**, permanent tables only, **not user-configurable or user-queryable**; recovery is done by Snowflake Support. |
| **Zero-Copy Cloning** | Instant copy of a table / schema / database | Clone shares the same micro-partitions via metadata pointers; storage is only consumed as the clone diverges. |

<br></br>

## 8. Security Architecture

* **Encryption:** All data is encrypted at rest (**AES-256**) and in transit (TLS), always on by default.
* **Hierarchical Key Model:** **Root key** (protected by a hardware security module, HSM) -> **Account master keys** -> **Table master keys** -> **File keys**. A separate key protects each level, limiting the blast radius of any single key.
* **Key Rotation & Re-keying:** Keys are rotated automatically (every 30 days); annual re-keying of data is available on Enterprise edition and above.
* **Customer-Managed Keys:** *Tri-Secret Secure* (Business Critical edition and above) combines a customer-held key with Snowflake's key.
* **Access Control:** RBAC (roles and privileges granted to roles), with optional Discretionary Access Control on object ownership.

<br></br>

## 9. Account, Deployment & Sharing Architecture

* **Object Hierarchy:** Organization -> Account -> Database -> Schema -> objects (tables, views, stages, etc.).
* **Account Deployment:** Each account lives on **one cloud provider in one region**. Compute and storage for that account are co-located in the region.
* **Secure Data Sharing:** Enabled by the metadata layer: a provider grants access to its micro-partitions through metadata, so **no data is copied or moved**. The consumer queries the shared data using **its own virtual warehouse** (and pays for its own compute). Cross-region / cross-cloud sharing uses replication.
* **Replication & Failover:** Databases (and account objects) can be replicated across regions and clouds for disaster recovery and business continuity.

<br></br>

## 10. How the Architecture Solved Legacy Bottlenecks

```
SHARED-DISK                           SHARED-NOTHING                        MULTI-CLUSTER SHARED-DATA
(e.g., Oracle RAC)                    (e.g., Teradata, Redshift Classic)   (Snowflake)

+------------------+                  +------------------+                 +------------------+
| Compute  Compute |                  | Compute  Compute |                 | Compute  Compute |  <-- Virtual Warehouse 1
+--------+---------+                  |  +Disk    +Disk  |                 +--------+---------+
         |                            +--------+---------+                          |
         | (Shared Interconnect)               | (Network Reshard)                  | (No Lock / Shared Access)
+--------+---------+                  +--------+---------+                 +--------+---------+
|   Shared Disk    |                  | Compute  Compute |                 | Compute  Compute |  <-- Virtual Warehouse 2
|   Subsystem      |                  |  +Disk    +Disk  |                 +--------+---------+
+------------------+                  +------------------+                          |
                                                                           +--------+---------+
                                                                           |  Cloud Object    |  <-- Centralized Storage
                                                                           |  Storage         |
                                                                           +------------------+

```

| Historical Paradigm | Architectural Limitation | Snowflake Architectural Solution |
| --- | --- | --- |
| **Shared-Disk** | Storage SAN/NAS interconnect creates disk I/O bottlenecks and complex network locking under scale. | **Stateless Compute + Object Storage:** Centralized cloud storage handles I/O scaling; independent Virtual Warehouses bypass network locking via metadata snapshots. |
| **Shared-Nothing** | Compute and storage are coupled. Adding storage requires buying compute. Re-balancing data requires heavy network re-sharding. | **Complete Decoupling:** Compute scales up/down instantly. Storage scales infinitely. Data is never explicitly re-sharded across cluster nodes. |
| **Single-Engine Concurrency** | Heavy ETL pipelines lock resources and starve real-time BI dashboards. | **Workload Isolation:** Dedicated Virtual Warehouses process ETL, Ad-hoc analytics, and BI workloads on separate compute clusters over identical data. |

<br></br>

## 11. Key Architectural Trade-Offs

### Advantages

* **Compute Workload Isolation:** No compute contention between ingestion, transformation, and reporting warehouses.
* **Instant Elasticity:** Compute can scale up, scale down, expand, or suspend within seconds without affecting running queries or storage.
* **Low Operational Overhead:** No manual indexing, partitioning strategies, distribution keys, or vacuuming required due to automated metadata pruning.
* **Pay-per-use Economics:** Storage and compute are billed independently; idle warehouses suspend and stop consuming credits.
* **Zero-copy features:** Cloning, Time Travel, and data sharing are cheap because of immutable micro-partitions plus metadata pointers.

### Disadvantages & Constraints

* **Higher Cold-Cache Latency:** Initial queries that bypass local SSD caches must retrieve micro-partitions from remote object storage over the network.
* **Storage-Compute Network Traversal:** Because compute nodes do not persist data, data must be transferred across cloud networks from object storage to compute caches, depending on cache hits.
* **Cache Loss on Suspend:** Suspending a warehouse clears its local cache, so aggressive auto-suspend can trade credit savings for slower first queries.
* **Table-level Write Serialization:** Concurrent `UPDATE` / `DELETE` / `MERGE` on the same table queue behind each other.
* **Cloud Dependency:** The architecture relies on public cloud object stores and virtualized infrastructure, so deployment is limited to supported cloud regions, and each account is bound to a single cloud and region.
