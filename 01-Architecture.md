# Snowflake Architecture
<br></br>
## 1. High-Level Architecture Overview

Snowflake utilizes a **Multi-Cluster Shared-Data Architecture**. It separates the database into three independently scalable layers: **Centralized Storage**, **Compute (Virtual Warehouses)**, and **Cloud Services**.

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
* **Storage Format:** Data is automatically converted into an optimized, **proprietary hybrid columnar format**.
* **Micro-Partitions:**
  * Physical data files containing between **50 MB to 500 MB** of uncompressed data.
  * Organized contiguously and stored as **immutable objects**.
  * Columns within each micro-partition are compressed individually using advanced encoding algorithms (e.g., Run-Length, Dictionary).
* **Key Mechanisms:**
  * **Immutability:** Updates and deletes do not overwrite existing files; instead, new micro-partition versions are created, enabling native **Time Travel** and **Zero-Copy Cloning**.
  * **Automatic Partitioning:** Data partitioning occurs automatically during ingestion without explicit partition keys.

### Layer 2: Compute Layer (Virtual Warehouses)

* **Physical Realization:** MPP (Massively Parallel Processing) compute clusters composed of virtual machine instances provisioned from the cloud provider (e.g., AWS EC2).
* **Stateless Processing:** Compute nodes do not persistently store table data. They fetch micro-partitions on-demand from the Centralized Storage layer into local high-speed SSDs (**Local Disk Caching**).
* **Isolation Mechanics:**
  * Virtual Warehouses do not share CPU, memory, or local storage.
  * Multiple independent warehouses can read from and write to the same underlying storage layer concurrently without locking contention.

* **Scaling Vectors:**
  * **Scale-Up (Vertical Elasticity):** Resizing a warehouse (e.g., `X-Small` to `4X-Large`) dynamically allocates more CPU/RAM per node to accelerate single, complex, resource-intensive queries.
  * **Scale-Out (Horizontal Elasticity):** Multi-Cluster Warehouses automatically spin up identical compute clusters behind a load balancer to scale query throughput during high concurrency.

### Layer 3: Cloud Services (The Control Plane)

* **Engine:** A distributed, stateless service layer backed by a transactional metadata store (FoundationDB).
* **Key Responsibilities:**
* **Metadata Management:** Tracks micro-partition file pointers, offsets, column min/max ranges, distinct values, and modification histories.
* **Query Optimization & Compilation:** Parses SQL, applies rule/cost-based optimizations using metadata, and generates execution plans for Virtual Warehouses.
* **Transaction Control:** Enforces ACID compliance across concurrent workloads using **Snapshot Isolation (MVCC)**.
* **Security & Governance:** Handles user authentication, RBAC (Role-Based Access Control), end-to-end encryption key management, and session states.

<br></br>

## 3. How the Architecture Solved Legacy Bottlenecks

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

## 4. Key Architectural Trade-Offs

### Advantages

* **Zero Resource Contention:** Complete workload isolation between ingestion, transformation, and reporting.
* **Instant Elasticity:** Compute can scale up, scale down, expand, or suspend within seconds without affecting running queries or storage.
* **Low Operational Overhead:** No manual indexing, partitioning strategies, distribution keys, or vacuuming required due to automated metadata pruning.

### Disadvantages & Constraints

* **Higher Cold-Cache Latency:** Initial queries that bypass local SSD caches must retrieve micro-partitions from remote object storage over the network.
* **Storage-Compute Network Traversal:** Because compute nodes are stateless, data must be transferred across cloud networks from object storage to compute caches, depending on cache hits.
* **Cloud Infrastructure Lock-in:** Architecture relies on public cloud object stores and virtualized infrastructure engines, requiring optimized deployment across supported cloud regions.
