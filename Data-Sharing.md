Snowflake Data Sharing allows organizations to share secure, read-only database objects (tables, dynamic tables, secure views, secure UDFs, external tables, and Apache Iceberg tables) across different Snowflake accounts, or with non-Snowflake users via Reader Accounts. Because it uses Snowflake's unique metadata architecture, data sharing happens instantly with **zero data copying, zero storage replication, and zero data movement fees**.

## 1. Core Architectural Mechanics

Data Sharing operates strictly at the cloud services metadata layer. The Data Provider grants access to live data blocks without moving physical micro-partitions from their storage location.

```
          [ Data Provider Account ]                  [ Data Consumer Account ]
         ┌─────────────────────────┐                ┌────────────────────────┐
         │ Production Storage Base │                │ Compute Warehouse only │
         │  (Immutable Partitions) │                │  (Queries Live Data)   │
         └────────────┬────────────┘                └───────────┬────────────┘
                      │                                         │
                      └─────────────── Shared Via ──────────────┘
                                  [ Secure Share ]
                             (Metadata Access Pointers)

```

* **Live Updates:** Data changes made by the Provider are immediately visible to Consumers in real-time. There is no refresh latency or scheduled sync required.
* **Decoupled Compute Costs:** The Provider pays for data storage. The Consumer pays for their own virtual warehouse compute resources used to query the shared data.
* **Read-Only Boundary:** Consumers can run complex analytical queries, joins, and clones on shared data, but they **cannot** perform DML operations (`INSERT`, `UPDATE`, `DELETE`, or `TRUNCATE`) on shared objects.

## 2. Secure Share Structural Workflow

Data sharing is managed via a specific first-class database object called a **Share**. A Share acts as an access control container wrapping the target database structures.

```
 [ Database ] --> [ Schema ] --> [ Secure Views / Tables ] --> Added to: [ SHARE ] --> Granted to: [ Consumer Account ]

```

### Setup Execution Sequence (Provider Side)

```sql
-- Step 1: Create an empty secure share container (Account-level object)
CREATE OR REPLACE SHARE sales_share
  COMMENT = 'Outgoing outbound share containing historical sales metrics';

-- Step 2: Grant structural usage boundaries to the share container
GRANT USAGE ON DATABASE prod_db TO SHARE sales_share;
GRANT USAGE ON SCHEMA prod_db.analytics TO SHARE sales_share;

-- Step 3: Append specific analytical assets to the share payload
GRANT SELECT ON TABLE prod_db.analytics.regional_sales TO SHARE sales_share;
GRANT SELECT ON SECURE VIEW prod_db.analytics.v_secure_customer_metrics TO SHARE sales_share;

-- Step 4: Authorize specific external Snowflake consumer accounts to mount the share
-- Best Practice: Use Modern Organization.Account format
ALTER SHARE sales_share ADD ACCOUNTS = myorg.xy12345, myorg.ab67890;

```

### Consumption Mount Sequence (Consumer Side)

```sql
-- Step 1: Discover available inbound shares exposed to your account
SHOW SHARES;

-- Step 2: Create a local reference database mapped directly to the inbound share
CREATE DATABASE shared_sales_analytics 
  FROM SHARE myorg.xy12345.sales_share; -- Formatted as [Organization].[Provider_Account].[Share_Name]

-- Step 3: Grant access to ALL objects inside the share simultaneously (The correct way)
GRANT IMPORTED PRIVILEGES ON DATABASE shared_sales_analytics TO ROLE data_analyst;

```

## 3. Sharing Topologies & Architectures

Snowflake provides three delivery patterns depending on the consumer's tech stack and licensing profile.

| Sharing Profile | Delivery Mechanism | Target Consumer Profile | Key Limitation / Characteristic |
| --- | --- | --- | --- |
| **Direct Sharing** | Account-to-Account Share | Existing Snowflake accounts within the **same cloud region**. | Fixed 1-to-1 or 1-to-many predefined static account target map. |
| **Snowflake Marketplace** | Public / Private Listings | Any global Snowflake customer. Supports monetization structures. | Governed by Snowflake Provider listings profiles and marketplace validation. |
| **Reader Accounts** | Managed Sub-Accounts | External clients/vendors who **do not own a Snowflake account**. | **Provider pays for the compute warehouse credits** consumed by the Reader. |

### Provisioning and Restricting a Client Reader Account

If an external partner needs your data but doesn't use Snowflake, you can spin up an isolated, single-tenant billing sub-account:

```sql
-- Step 1: Create the managed consumer web portal sub-account
CREATE MANAGED ACCOUNT client_abc_reader
  ADMIN_NAME = 'client_admin',
  ADMIN_PASSWORD = 'TemporaryPassword123!',
  TYPE = READER,
  COMMENT = 'Dedicated reader portal for external vendor ABC tracking';

-- Step 2: Link your share container to the newly spun up reader account ID
-- Run SHOW MANAGED ACCOUNTS to grab the auto-generated account locator string
ALTER SHARE sales_share ADD ACCOUNTS = reader_account_locator_id;

```

* **Reader Restriction:** Reader accounts are entirely read-only. Users in a reader account *cannot* perform any Data Manipulation Language (DML) tasks, such as loading data or running `INSERT`/`UPDATE` operations, and can only consume data from the single provider account that spun them up.

## 4. The Secure View Mandate & Performance Tuning

### The Standard View Exposure Risk

Never add a standard `CREATE VIEW` asset to a public or direct Share. Standard views allow the query planner to expose underlying transformation errors and index values inside optimizing filters. A malicious consumer could exploit these error messages to infer records outside their authorized filter range.

### Creating Filtered Multi-Tenant Secure Views

Use the `SECURE` keyword combined with context functions (`CURRENT_ACCOUNT()`) to dynamically filter data based on who is querying the share.

```sql
CREATE OR REPLACE SECURE VIEW prod_db.analytics.v_secure_customer_metrics AS
SELECT 
    customer_id,
    account_locator_reference,
    sales_revenue_usd,
    margin_percentage
FROM prod_db.analytics.base_metrics
-- Dynamically matches the exact Snowflake account string of the active consumer session
WHERE account_locator_reference = CURRENT_ACCOUNT();

```

### The Pruning Optimization Tradeoff

* **Performance Impact:** Secure views intentionally disable certain query optimizer heuristics (like filter pushdowns) to protect data privacy. If your shared secure view joins massive tables, query execution might slow down significantly for the consumer.
* **Fix:** Cluster the underlying base tables explicitly on the keys used in the secure view filter (e.g., `account_locator_reference`) to ensure efficient **partition pruning** even through the secure boundary.

## 5. Cross-Database Secure Views (`REFERENCE_USAGE`)

When a shared secure view references tables or views residing across **multiple databases** within your provider account, a standard database `USAGE` grant will fail. Because a Share maps directly to a primary database on the consumer end, you must explicitly grant a special dependency permission called `REFERENCE_USAGE` on any secondary databases involved.

```sql
-- Step 1: Create a view in your primary database that references a secondary database
CREATE OR REPLACE SECURE VIEW prod_db.analytics.v_global_sales_inventory AS
SELECT 
    s.customer_id,
    s.product_id,
    s.sales_revenue_usd,
    i.warehouse_stock_count  -- References an external database
FROM prod_db.analytics.regional_sales s
JOIN supply_chain_db.inventory.product_stock i 
  ON s.product_id = i.product_id;

-- Step 2: Grant standard usage to the primary database hosting the view
GRANT USAGE ON DATABASE prod_db TO SHARE sales_share;
GRANT USAGE ON SCHEMA prod_db.analytics TO SHARE sales_share;

-- Step 3: Grant REFERENCE_USAGE to the external dependency database
-- Note: You do NOT need to grant schema or table level access inside supply_chain_db
GRANT REFERENCE_USAGE ON DATABASE supply_chain_db TO SHARE sales_share;

-- Step 4: Add the secure view payload to the share
GRANT SELECT ON VIEW prod_db.analytics.v_global_sales_inventory TO SHARE sales_share;

```

* **Implicit Pass-Through:** The consumer does not get direct visibility or query privileges into `supply_chain_db`. Access to its data is strictly granted as an encrypted pass-through context while executing that specific secure view.
* **Database Role Restriction:** The `REFERENCE_USAGE` privilege must be granted directly to the account-level Share container object. It **cannot** be granted to a Snowflake Database Role.

## 6. Metadata Object Restrictions & Behavior

Data Sharing enforces a strict security perimeter. Certain object types transform or fail entirely when wrapped in a Share.

| Object Reference | Allowed in Share? | Functional Behavior / Failure Mode |
| --- | --- | --- |
| **Standard Tables** | **Yes** | Fully available. Live data updates stream directly to consumer. |
| **Dynamic Tables** | **Yes** | Incremental data additions refresh on your schedule; consumers query the pre-computed state. |
| **Standard Views** | **Blocked** | Dropping a non-secure view into a share container triggers a compilation block. |
| **Streams (CDC)** | **Blocked** | You cannot share a Stream object directly. However, you can host a stream *on* a shared table at the consumer end to track inbound deltas. |
| **Cloned Objects** | **Yes** | A cloned table can be shared, but it is treated as a static snapshot at the moment of the clone. |
| **Shared Data** | **Blocked** | Traditional multi-hop sharing is blocked. A consumer **cannot re-share** a standard database mounted from an inbound share without advanced Resharing configurations. |
| **Open Table Formats** | **Yes** | Both managed and externally managed **Apache Iceberg tables** can be shared directly via Zero-Copy architectures. |
| **Semantic Views** | **Yes** | Pre-configured business-metric semantic views can be explicitly shared as first-class objects. |


## 7. Advanced Patterns: Zero-ETL "Resharing"

To address scenarios where a business unit needs to add local logic, column aliases, or security policies to an upstream dataset and pass it on, Snowflake supports **Resharing**.

This allows an intermediary account to consume a share, create downstream secure views or wrappers over it, and expose that derivative dataset to a third account **without materializing or copying the underlying data to physical local storage**.

```
[Original Provider] ──(Share 1)──> [Intermediary Consumer] ──(Apply Logic/Views)──(Share 2)──> [Downstream Consumer]

```

## 8. Compliance, Governance & Cross-Edition Sharing

### Account Edition Incompatibilities (Business Critical Wall)

By default, Snowflake blocks sharing data from a **Business Critical** (or higher) account to a lower, **Non-Business Critical** account to prevent accidental exposure of highly regulated (e.g., HIPAA, PCI-DSS) data.

```sql
-- Step 1: Grant the override privilege to a targeted custom role if needed
USE ROLE ACCOUNTADMIN;
GRANT OVERRIDE SHARE RESTRICTIONS ON ACCOUNT TO ROLE data_governance_admin;

-- Step 2: Apply the bypass directly while assigning the non-BC consumer account
USE ROLE data_governance_admin;
ALTER SHARE sales_share ADD ACCOUNTS = myorg.non_bc_account SHARE_RESTRICTIONS=false;

```

### Governance and Listing Observability

| Monitoring View | System Path | Purpose |
| --- | --- | --- |
| **Access History** | `SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY` | Audits DDL/DML tracking on shared assets (`CREATE`, `ALTER`, `DROP` on listings). |
| **Sharing History** | `SNOWFLAKE.ACCOUNT_USAGE.DATA_SHARING_USAGE_HISTORY` | Tracks precisely which consumer accounts are querying your listings, warehouses, and metrics. |

```sql
-- Audit exactly which consumer accounts are actively querying your shared assets
SELECT 
    SHARE_NAME,
    CONSUMER_ACCOUNT_NAME,
    SCHEMA_NAME,
    TABLE_NAME,
    COUNT(1) AS TOTAL_READ_OPERATIONS,
    SUM(BYTES_SCANNED) AS TOTAL_DATA_VOLUME_SCANNED
FROM SNOWFLAKE.ACCOUNT_USAGE.DATA_SHARING_USAGE_HISTORY
WHERE QUERY_START_TIME >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1, 2, 3, 4
ORDER BY TOTAL_DATA_VOLUME_SCANNED DESC;

```

## 9. Business Continuity & Disaster Recovery (BCDR)

For shared data products published via Snowflake Listings, providers can leverage **Listing Business Continuity and Disaster Recovery (Listing BCDR)**.

If the primary cloud infrastructure region undergoes a major outage, automated failover rules redirect consumer connection strings to a replicated secondary data share instance in another region or cloud provider. This ensures consumer production pipelines, analytics applications, and AI workloads continue running uninterrupted without manual backend re-pointing.


## 10. Critical Production Pitfalls & Anti-Patterns

* **The Cross-Region Sharing Block:** Direct sharing is strictly bounded to accounts residing in the **same cloud infrastructure region** (e.g., AWS us-east-1 to AWS us-east-1). To share data across regions or cloud providers (e.g., AWS to Azure), you must use **Snowflake Business Continuity Replication Groups** to replicate the base database to that target region before creating a local share.
* **The Shared Object Rename Disruption:** If you drop, recreate (`CREATE OR REPLACE`), or rename a table or view that is currently attached to an active Share, the binding breaks immediately. The consumer will hit an unexpected object compilation fault (`Object does not exist`) on their pipeline runs.
* **The Multi-Database View Dependency:** If a secure view added to a share targets tables residing across multiple databases, you must explicitly grant `REFERENCE_USAGE` to the share container for **every external database** targeted. If the primary database is replicated across regions, all secondary dependency databases referenced via `REFERENCE_USAGE` must belong to that exact same Replication Group or replication will fail.
* **Reader Account Credit Drain:** Reader Accounts are funded completely by the Provider's credit pool. If an external client runs poorly optimized, unindexed queries on a Reader Account using a warehouse with no auto-suspend limits, **it will continuously consume your organization's credits**. Always configure a strict resource monitor on reader accounts:
```sql
CREATE RESOURCE MONITOR reader_guard_rails WITH CREDIT_QUOTA = 50
  TRIGGERS ON 100 PERCENT DO SUSPEND;
ALTER ACCOUNT client_abc_reader SET RESOURCE_MONITOR = reader_guard_rails;

```
