## 1. Concepts at a Glance

| Concept | What It Is | Example |
| --- | --- | --- |
| **Parameter** | A named setting that controls Snowflake behaviour and can be set at one or more **levels** (account, user, session, object). Lower levels can override higher ones | `STATEMENT_TIMEOUT_IN_SECONDS`, `TIMEZONE`, `DATA_RETENTION_TIME_IN_DAYS` |
| **Object property** | A fixed attribute of one object, set in its `CREATE` / `ALTER`. Not inherited, no hierarchy | Warehouse `AUTO_SUSPEND`, `WAREHOUSE_SIZE`; user `DEFAULT_ROLE` |
| **Session variable** | A user-defined variable for one session (`SET my_var = 1`) | `SET env = 'DEV'` |
| **Role** | A named container of privileges, granted to users and to other roles | `SYSADMIN`, `ANALYST` |
| **Privilege** | A specific permission on a securable object | `SELECT` on a table, `USAGE` on a warehouse |
| **Role hierarchy** | Roles granted to roles, so the parent **inherits** the child's privileges | `ANALYST` granted to `SYSADMIN` |

**Quick distinction:** *Parameters* answer "how should things behave?" (with override levels). *Roles and privileges* answer "who may do what?" (with inheritance through the role hierarchy). Parameters are **never** attached to roles; they attach to the account, a user, a session, or an object.

<br></br>

# PART A: PARAMETERS

## 2. Parameter Types

| Type | Can Be Set At | Purpose | Examples |
| --- | --- | --- | --- |
| **Account parameters** | **Account only** | Account-wide security, integration, and platform behaviour | `MIN_DATA_RETENTION_TIME_IN_DAYS`, `PERIODIC_DATA_REKEYING`, `REQUIRE_STORAGE_INTEGRATION_FOR_STAGE_CREATION`, `PREVENT_UNLOAD_TO_INLINE_URL`, `SAML_IDENTITY_PROVIDER` |
| **Session parameters** | **Account, User, Session** | How SQL behaves in a session: formats, timeouts, caching, time zone | `TIMEZONE`, `DATE_OUTPUT_FORMAT`, `USE_CACHED_RESULT`, `QUERY_TAG`, `AUTOCOMMIT`, `LOCK_TIMEOUT`, `STATEMENT_TIMEOUT_IN_SECONDS` |
| **Object parameters** | **Account, and the object levels** (database, schema, table, warehouse, task, pipe, etc.) | Defaults for objects, inheritable down the object hierarchy | `DATA_RETENTION_TIME_IN_DAYS`, `DEFAULT_DDL_COLLATION`, `MAX_CONCURRENCY_LEVEL`, `SUSPEND_TASK_AFTER_NUM_FAILURES` |

Some parameters belong to **more than one family**. For example, `STATEMENT_TIMEOUT_IN_SECONDS` and `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS` are session parameters **and** can be set on a warehouse. A few, such as `NETWORK_POLICY`, can be set at account and user level.

<br></br>

## 3. Parameter Hierarchy and Precedence

### 3.1 The General Rule

> **The most specific level that has an explicit value wins.** If a level has no explicit value, it **inherits** from the next level up. If nothing sets a value, the **Snowflake default** applies.

### 3.2 Session Parameter Hierarchy

```
 SNOWFLAKE DEFAULT
        │  (if not set)
        ▼
 ACCOUNT   ALTER ACCOUNT SET ...        applies to everyone
        │  overridden by
        ▼
 USER      ALTER USER <u> SET ...       applies to that user's sessions
        │  overridden by
        ▼
 SESSION   ALTER SESSION SET ...        applies until the session ends
           (or set by the driver / connection string)
```

| Level Set | Effective Value for the Session |
| --- | --- |
| Nothing set | Snowflake default |
| Account only | Account value |
| Account + User | **User** value |
| Account + User + Session | **Session** value |
| After `ALTER SESSION UNSET` | Falls back to **User** value, else **Account** value, else default |

### 3.3 Object Parameter Hierarchy

Object parameters follow the **object containment hierarchy**:

```
 ACCOUNT
   └── DATABASE
         └── SCHEMA
               └── TABLE  (also other schema objects that support the parameter)
```

* A value set on a container is **inherited** by everything inside it that has no value of its own.
* A value set on a lower object **overrides** the higher one for that object (and anything under it).

**Worked example: `DATA_RETENTION_TIME_IN_DAYS`**

| Level | Explicit Value | Effective for |
| --- | --- | --- |
| Account | 1 | Everything not overridden below |
| Database `PROD` | 14 | All schemas/tables in `PROD` with no own value |
| Schema `PROD.RAW` | 3 | Tables in `PROD.RAW` with no own value |
| Table `PROD.RAW.ORDERS` | 30 | Only that table |

Result: `ORDERS` = 30, other `RAW` tables = 3, other `PROD` schemas' tables = 14, tables outside `PROD` = 1.

### 3.4 Warehouse Parameters

Warehouse-scoped parameters (`MAX_CONCURRENCY_LEVEL`, `STATEMENT_TIMEOUT_IN_SECONDS`, `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS`) can be set at **account** and on the **warehouse** (the two timeouts also at user/session).

* For the two timeouts, when a value is set on **both** the warehouse and the session, the **lower (non-zero) value is enforced**. Use this to let a platform team cap runaway queries on a warehouse regardless of session settings.

### 3.5 Special Cases and Exceptions to "Lowest Level Wins"

| Parameter | Special Behaviour |
| --- | --- |
| `MIN_DATA_RETENTION_TIME_IN_DAYS` (account only) | Acts as a **floor**: effective retention = **greater of** the object's `DATA_RETENTION_TIME_IN_DAYS` and the account minimum. A table set to 1 day under a minimum of 7 effectively gets 7 |
| `STATEMENT_TIMEOUT_IN_SECONDS` / `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS` | Warehouse and session values both apply; the **lowest non-zero** wins (see 3.4) |
| `NETWORK_POLICY` | Set on account and/or user; the **user-level** policy overrides the account policy for that user |
| Account-only parameters | No hierarchy; one value for the whole account |
| `DATA_RETENTION_TIME_IN_DAYS` limits | Max 1 day on Standard edition; up to 90 days on Enterprise and above for permanent tables; transient and temporary objects max 1 day |

Other account-level policies (password, authentication, session policies) follow a similar **account then user override** idea, but they are *policy objects*, not parameters.

### 3.6 Inheritance Is Evaluated by Level, Not Copied

An object with **no explicit value** reflects whatever its nearest ancestor currently has. Use `SHOW PARAMETERS` and its `level` column (next section) to see where an effective value actually comes from.

<br></br>

## 4. Setting, Viewing, and Resetting Parameters

### 4.1 Setting and Unsetting

```sql
-- Account level (affects everyone not overridden)
ALTER ACCOUNT SET STATEMENT_TIMEOUT_IN_SECONDS = 7200;
ALTER ACCOUNT UNSET STATEMENT_TIMEOUT_IN_SECONDS;

-- User level
ALTER USER alice SET TIMEZONE = 'Asia/Kolkata', STATEMENT_TIMEOUT_IN_SECONDS = 3600;
ALTER USER alice UNSET TIMEZONE;

-- Session level (ends with the session)
ALTER SESSION SET TIMEZONE = 'UTC', USE_CACHED_RESULT = FALSE;
ALTER SESSION UNSET USE_CACHED_RESULT;

-- Object levels
ALTER DATABASE prod SET DATA_RETENTION_TIME_IN_DAYS = 14;
ALTER SCHEMA prod.raw SET DATA_RETENTION_TIME_IN_DAYS = 3;
ALTER TABLE prod.raw.orders SET DATA_RETENTION_TIME_IN_DAYS = 30;
ALTER WAREHOUSE bi_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 1800,
                          STATEMENT_QUEUED_TIMEOUT_IN_SECONDS = 300;

-- Can also be set at creation
CREATE SCHEMA prod.staging DATA_RETENTION_TIME_IN_DAYS = 0;   -- where the edition allows
```

`UNSET` removes the explicit value so the object **inherits again**; setting a value equal to the default is not the same as unsetting.

### 4.2 Viewing Parameters

```sql
SHOW PARAMETERS;                                          -- session-level view
SHOW PARAMETERS LIKE 'STATEMENT%' IN SESSION;
SHOW PARAMETERS IN ACCOUNT;
SHOW PARAMETERS IN USER alice;
SHOW PARAMETERS LIKE 'DATA_RETENTION%' IN DATABASE prod;
SHOW PARAMETERS LIKE 'DATA_RETENTION%' IN TABLE prod.raw.orders;
SHOW PARAMETERS IN WAREHOUSE bi_wh;
```

| Output Column | Meaning |
| --- | --- |
| `key` | Parameter name |
| `value` | **Effective** value in this scope |
| `default` | Snowflake default |
| `level` | **Where the effective value is set**: empty (default), `ACCOUNT`, `USER`, `SESSION`, `DATABASE`, `SCHEMA`, `TABLE`, `WAREHOUSE`, etc. |
| `description` / `type` | Description and data type |

**Reading `level` is the fastest way to debug precedence.** If `level` says `USER` when you expected `SESSION`, nothing has been set at the session level.

### 4.3 Who Can Set What

| Scope | Typical Privilege Needed |
| --- | --- |
| **Account** | `ACCOUNTADMIN` (some parameters can be delegated through specific global privileges) |
| **User** | Owner of the user (commonly `USERADMIN` / `SECURITYADMIN`) or `ACCOUNTADMIN`; users can change some of their own settings |
| **Session** | Any user, for their own session |
| **Database / Schema / Table** | `OWNERSHIP` on the object (or `MODIFY` where applicable) |
| **Warehouse** | `MODIFY` or `OWNERSHIP` on the warehouse |

Because parameter changes are privileged operations, they tie into the role design in Part B.

<br></br>

## 5. Key Parameters Reference

*Defaults are typical values; confirm in your account with `SHOW PARAMETERS`.*

### 5.1 Session Parameters (Account / User / Session)

| Parameter | Default | Purpose |
| --- | --- | --- |
| `STATEMENT_TIMEOUT_IN_SECONDS` | 172800 (2 days) | Cancel statements running longer than this |
| `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS` | 0 (no limit) | Cancel statements queued longer than this |
| `LOCK_TIMEOUT` | 43200 (12 hours) | How long a statement waits for a lock |
| `USE_CACHED_RESULT` | TRUE | Allow result cache reuse |
| `TIMEZONE` | `America/Los_Angeles` | Session time zone (affects `TIMESTAMP_LTZ`, `CURRENT_TIMESTAMP`) |
| `TIMESTAMP_TYPE_MAPPING` | `TIMESTAMP_NTZ` | What plain `TIMESTAMP` means |
| `DATE_OUTPUT_FORMAT` / `TIMESTAMP_OUTPUT_FORMAT` | `YYYY-MM-DD` / `YYYY-MM-DD HH24:MI:SS.FF3 TZHTZM` | Output formatting |
| `AUTOCOMMIT` | TRUE | Each statement commits automatically |
| `TRANSACTION_DEFAULT_ISOLATION_LEVEL` | `READ COMMITTED` | Isolation level for transactions |
| `QUERY_TAG` | empty | Label queries for monitoring and cost attribution |
| `QUOTED_IDENTIFIERS_IGNORE_CASE` | FALSE | Treat quoted identifiers case-insensitively |
| `WEEK_START` / `WEEK_OF_YEAR_POLICY` | 0 / 0 | Week calculation behaviour |
| `ROWS_PER_RESULTSET` | 0 (unlimited) | Cap rows returned |
| `MULTI_STATEMENT_COUNT` | 1 | Allow multiple statements per request |
| `ABORT_DETACHED_QUERY` | FALSE | Abort queries if the client disconnects |
| `CLIENT_SESSION_KEEP_ALIVE` | FALSE | Keep sessions alive past the idle timeout |
| `ERROR_ON_NONDETERMINISTIC_MERGE` | TRUE | Error on ambiguous `MERGE` matches |

### 5.2 Object Parameters

| Parameter | Levels | Default | Purpose |
| --- | --- | --- | --- |
| `DATA_RETENTION_TIME_IN_DAYS` | Account, Database, Schema, Table | 1 | Time Travel retention |
| `MAX_DATA_EXTENSION_TIME_IN_DAYS` | Account, Database, Schema, Table | 14 | Max extension of retention to keep streams from going stale |
| `DEFAULT_DDL_COLLATION` | Account, Database, Schema, Table | empty | Default collation for new columns |
| `MAX_CONCURRENCY_LEVEL` | Account, Warehouse | 8 | Target concurrent statements per warehouse cluster |
| `STATEMENT_TIMEOUT_IN_SECONDS` / `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS` | Account, Warehouse (plus user/session) | see 5.1 | Timeouts for work on that warehouse |
| `SUSPEND_TASK_AFTER_NUM_FAILURES` | Account, Database, Schema, Task | 0 (never) | Auto-suspend a task after consecutive failures |
| `PIPE_EXECUTION_PAUSED` | Account, Schema, Pipe | FALSE | Pause Snowpipe ingestion |
| `LOG_LEVEL` / `TRACE_LEVEL` / `METRIC_LEVEL` | Account and several object types | varies | Telemetry verbosity |

### 5.3 Account-Only Parameters (Examples)

| Parameter | Purpose |
| --- | --- |
| `MIN_DATA_RETENTION_TIME_IN_DAYS` | Floor for Time Travel retention across the account |
| `PERIODIC_DATA_REKEYING` | Annual re-keying of data (Enterprise and above) |
| `REQUIRE_STORAGE_INTEGRATION_FOR_STAGE_CREATION` / `..._OPERATION` | Force stages to use storage integrations (no raw credentials) |
| `PREVENT_UNLOAD_TO_INLINE_URL` | Block unloading to ad-hoc URLs |
| `PREVENT_UNLOAD_TO_INTERNAL_STAGES` | Block unloading to internal stages |
| `NETWORK_POLICY` | Account-wide allowed IPs (user-level policy can override) |
| `SAML_IDENTITY_PROVIDER` / `SSO_LOGIN_PAGE` | SSO configuration |

<br></br>

## 6. Practical Parameter Scenarios

| Goal | Approach |
| --- | --- |
| Stop runaway ad-hoc queries without affecting ETL | Set `STATEMENT_TIMEOUT_IN_SECONDS` on the **ad-hoc warehouse** (e.g., 1800); leave the ETL warehouse at a high value |
| Give one analyst a local time zone | `ALTER USER alice SET TIMEZONE = 'Asia/Kolkata'` (user-level override of account default) |
| Make benchmarks honest | `ALTER SESSION SET USE_CACHED_RESULT = FALSE` for the test session only |
| Short retention for staging, long for finance | Set `DATA_RETENTION_TIME_IN_DAYS` on the staging schema (low) and finance database (high); keep account default as the baseline |
| Guarantee a company-wide minimum retention | `ALTER ACCOUNT SET MIN_DATA_RETENTION_TIME_IN_DAYS = 7` |
| Attribute cost by job | `ALTER SESSION SET QUERY_TAG = 'team=finance;job=nightly'` in each pipeline |
| Cap queued waiting on a BI warehouse | `ALTER WAREHOUSE bi_wh SET STATEMENT_QUEUED_TIMEOUT_IN_SECONDS = 120` |
| Per-role behaviour (e.g., stricter timeout for `ANALYST`) | **Not possible directly**: parameters don't attach to roles. Use **user-level** parameters, a **dedicated warehouse** with its own limits, or resource monitors |

<br></br>

## 7. Parameter Pitfalls

* Confusing **properties** with **parameters** (e.g., `AUTO_SUSPEND` is a warehouse property and has no hierarchy).
* Changing an **account-level** parameter and unintentionally affecting every user and job.
* Forgetting that a **user-level** value silently overrides the account value, so an account change "does nothing" for that user.
* Setting `ALTER SESSION` values and expecting them to persist: they end with the session (BI tools and connection pools often start fresh sessions).
* Not reading the `level` column when debugging precedence.
* Setting `DATA_RETENTION_TIME_IN_DAYS` low on a table while `MIN_DATA_RETENTION_TIME_IN_DAYS` silently raises it.
* Setting a timeout only at session level and assuming it caps work on a shared warehouse (the warehouse value can be lower and wins).
* Expecting retention values above the **edition limit** (1 day on Standard).
* Setting a parameter back to its default value instead of using `UNSET`: an explicit value still overrides what the level above would provide; use `UNSET` to inherit again.

<br></br>

# PART B: ROLES AND ROLE HIERARCHY

## 8. The Access Control Model

Snowflake combines two models:

| Model | Idea in Snowflake |
| --- | --- |
| **DAC** (Discretionary Access Control) | Every object has an **owner role** that can grant access to it to other roles |
| **RBAC** (Role-Based Access Control) | **Privileges are granted to roles**, and **roles are granted to users (or other roles)**. Users never hold privileges directly |

```
 USER ──(granted)──► ROLE ──(granted privileges)──► SECURABLE OBJECT
                       │
                       └──(granted role)──► another ROLE (hierarchy)
```

**Key facts**

* A **securable object** is anything access can be granted on (account, user, role, warehouse, database, schema, table, view, stage, task, etc.).
* Every object has exactly **one owner role** (the role with `OWNERSHIP`). Ownership is a privilege held by a role, not a user.
* A user can be granted **many roles**, but in a session has one **primary role** plus optional **secondary roles** active.
* Access is **denied by default**: a role can do only what it (or a role it inherits) has been granted.

<br></br>

## 9. Role Types

| Role Type | Scope | Typical Use |
| --- | --- | --- |
| **Account roles** (system-defined and custom) | Whole account | Users' working roles, administration, ownership |
| **Database roles** | A single database | Bundle privileges on that database's objects; portable with the database (replication, **sharing**) |
| **Application roles** | Inside a Snowflake Native App | Control access to an app's objects |
| **Instance roles** | An instance of a class (e.g., ML classification) | Control access to that instance's methods |
| **Snowflake-provided database roles** | In the `SNOWFLAKE` database | Delegate access to Snowflake metadata and usage data |

**Snowflake-provided database roles (examples):** `SNOWFLAKE.USAGE_VIEWER` (usage / cost views), `SNOWFLAKE.GOVERNANCE_VIEWER` (governance views), `SNOWFLAKE.OBJECT_VIEWER` (object metadata views), `SNOWFLAKE.SECURITY_VIEWER` (security-related views), and Cortex roles such as `SNOWFLAKE.CORTEX_USER`. These let you grant read access to `ACCOUNT_USAGE` data **without** giving away broad admin roles.

<br></br>

## 10. System-Defined Roles

```
                 ORGADMIN   (organization level, separate from the account hierarchy)

                 ACCOUNTADMIN
                  ├────────────────────────┐
             SECURITYADMIN              SYSADMIN
                  │                         │
              USERADMIN              (custom roles roll up here)
                  │                         │
                  └───────────┬─────────────┘
                           PUBLIC
```

| Role | Primary Responsibility | Key Capabilities |
| --- | --- | --- |
| **ORGADMIN** | Manage the **organization** | Create and manage accounts in the organization, view organization-level usage, enable replication for accounts. Not part of the account's role chain |
| **ACCOUNTADMIN** | Top-level **account** role | Encapsulates `SYSADMIN` and `SECURITYADMIN`. Manages account-level settings, parameters, billing and resource monitors, and has all privileges by inheritance |
| **SECURITYADMIN** | **Security and grants** | Holds the global `MANAGE GRANTS` privilege: can grant or revoke any privilege on any object, and manage roles and users. Inherits `USERADMIN` |
| **USERADMIN** | **Users and roles** | Has `CREATE USER` and `CREATE ROLE`; creates and manages users and roles it owns |
| **SYSADMIN** | **Objects and compute** | Can create warehouses, databases, and other objects. Intended to **own** most objects and to sit above all custom roles |
| **PUBLIC** | **Everyone** | Automatically granted to every user and every role. Anything granted to `PUBLIC` is accessible to all |

**Rules of thumb**

* `ACCOUNTADMIN` is for **account administration only**, not daily work, and should **not own objects**.
* Create users/roles with `USERADMIN`; manage grants with `SECURITYADMIN`; create objects with `SYSADMIN` (or a custom role under it).
* Don't put data privileges on `PUBLIC` unless you truly mean "everyone, including future users".

<br></br>

## 11. Role Hierarchy and Privilege Inheritance

### 11.1 How Inheritance Works

When role **A** is granted to role **B** (`GRANT ROLE A TO ROLE B`):

* **B is the parent** and **inherits all privileges of A** (including A's own inherited privileges).
* **A does not gain** B's privileges. Inheritance flows **upward** to parents.
* Anyone using B effectively holds A's and B's privileges.

```
 GRANT ROLE analyst TO ROLE sysadmin;     -- analyst is the CHILD, sysadmin the PARENT

           SYSADMIN      <- inherits everything below
              │
           ANALYST       <- child
```

**Mnemonic:** `GRANT ROLE <child> TO ROLE <parent>`. Privileges **roll up** the tree.

### 11.2 Multi-Level Example

```
 SYSADMIN
   ├── DATA_ENGINEER ──► SALES_WRITE ──► SALES_READ
   └── ANALYST ───────► SALES_READ
```

| Role | Effective Privileges |
| --- | --- |
| `SALES_READ` | Read on `SALES` |
| `SALES_WRITE` | Write on `SALES` **plus** everything in `SALES_READ` |
| `ANALYST` | Whatever it is directly granted **plus** `SALES_READ` |
| `DATA_ENGINEER` | Directly granted **plus** `SALES_WRITE` and `SALES_READ` |
| `SYSADMIN` | Everything above, since both `DATA_ENGINEER` and `ANALYST` roll up to it |

### 11.3 Hierarchy Rules and Properties

| Rule | Detail |
| --- | --- |
| **No circular grants** | A role cannot (directly or indirectly) be granted to itself |
| **Many-to-many** | A role can have multiple parents and multiple children (a DAG, not just a tree) |
| **Granting a role needs authority** | The granting role must **own** the role or hold `MANAGE GRANTS` |
| **Revoking breaks inheritance** | `REVOKE ROLE child FROM ROLE parent` removes the inherited privileges from the parent (and its ancestors, unless inherited via another path) |
| **Ownership is not inherited by "use"** | A parent can use a child's privileges, but objects it creates are owned by the **active primary role**, not by the child |
| **Inherited privileges include ownership-based access** | A parent of an owner role can manage that owner role's objects through inheritance |
| **System roles' hierarchy is fixed** | `ACCOUNTADMIN` > `SECURITYADMIN` > `USERADMIN`, `ACCOUNTADMIN` > `SYSADMIN`, all > `PUBLIC` |

**Why custom roles must roll up to `SYSADMIN`:** Roles created by `USERADMIN` are separate from the `SYSADMIN` branch until granted. If a custom role owns objects and is **not** under `SYSADMIN`, then `SYSADMIN` (and by inheritance a normal admin) **cannot see or manage** those objects. Rolling custom roles up to `SYSADMIN` keeps administration complete.

<br></br>

## 12. Active Roles: Primary, Secondary, and Defaults

### 12.1 Primary Role

* The role the session is currently **using**: `USE ROLE analyst;`
* Determines **who owns objects you create** and is the role evaluated for `CREATE <object>` authorization.
* Each user has a **default role** (`DEFAULT_ROLE`) used at login. If none is set, `PUBLIC`.

### 12.2 Secondary Roles

* Additional roles active at the same time (`USE SECONDARY ROLES ALL;`).
* Access is evaluated against the **union** of privileges from the primary role and all secondary roles (and everything each inherits).
* **Secondary roles cannot authorize `CREATE <object>` statements**, and newly created objects are owned by the **primary** role only.
* Controlled per user with `DEFAULT_SECONDARY_ROLES`.

```sql
USE ROLE analyst;                       -- primary role
USE SECONDARY ROLES ALL;                -- add all other granted roles' privileges
USE SECONDARY ROLES role_a, role_b;     -- only specific roles
USE SECONDARY ROLES NONE;               -- primary role only

ALTER USER alice SET DEFAULT_ROLE = analyst,
                     DEFAULT_SECONDARY_ROLES = ('ALL');
```

**Why secondary roles matter:** a user needing to join data owned by two different functional roles doesn't have to switch roles or combine them into a mega-role; with secondary roles both sets of privileges are available together.

### 12.3 Context Functions

| Function | Returns |
| --- | --- |
| `CURRENT_ROLE()` | The active primary role |
| `CURRENT_SECONDARY_ROLES()` | Active secondary roles |
| `CURRENT_AVAILABLE_ROLES()` | All roles granted to the user |
| `IS_ROLE_IN_SESSION('ROLE_NAME')` | TRUE if the role is active in the session, **including via the hierarchy**. Commonly used in masking and row access policies |

### 12.4 Stored Procedures and Roles

* **Owner's rights** procedures run with the **owner's** privileges and role context.
* **Caller's rights** procedures run with the **caller's** active role, so they can do only what the caller can.

<br></br>

## 13. Privileges in Depth

### 13.1 Privilege Categories

| Category | Examples |
| --- | --- |
| **Global (account-level)** | `CREATE WAREHOUSE`, `CREATE DATABASE`, `CREATE USER`, `CREATE ROLE`, `MANAGE GRANTS`, `MONITOR USAGE`, `MONITOR EXECUTION`, `EXECUTE TASK`, `CREATE INTEGRATION`, `APPLY MASKING POLICY`, `APPLY ROW ACCESS POLICY`, `APPLY TAG` |
| **Container usage** | `USAGE` on database, schema |
| **Object access** | `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `REFERENCES` on tables; `SELECT` on views and streams; `USAGE` on file formats, sequences, functions, procedures |
| **Object creation** | `CREATE SCHEMA` (on database); `CREATE TABLE`, `CREATE VIEW`, `CREATE STAGE`, `CREATE TASK`, etc. (on schema) |
| **Operational** | Warehouse: `USAGE`, `OPERATE`, `MONITOR`, `MODIFY`. Task/pipe: `OPERATE`, `MONITOR` |
| **Special** | `OWNERSHIP`, `ALL [PRIVILEGES]`, `IMPORTED PRIVILEGES` (on shared databases and `SNOWFLAKE`) |

### 13.2 The Access Path Rule

To use an object, a role needs privileges on **every container above it** plus the object itself:

| Action | Privileges Required |
| --- | --- |
| `SELECT` from a table | `USAGE` on **database**, `USAGE` on **schema**, `SELECT` on **table**, and `USAGE` on a **warehouse** |
| `INSERT` into a table | `USAGE` on database and schema, `INSERT` on table, `USAGE` on warehouse |
| Create a table | `USAGE` on database and schema, `CREATE TABLE` on schema |
| Create a schema | `USAGE` on database, `CREATE SCHEMA` on database |
| `COPY INTO` a table from a stage | `INSERT` on table, `READ` on internal stage (or `USAGE` on external stage), `USAGE` on file format, `USAGE` on warehouse |
| Query `ACCOUNT_USAGE` views | `IMPORTED PRIVILEGES` on `SNOWFLAKE` (or a `SNOWFLAKE` database role) and `USAGE` on a warehouse |

A frequent bug: granting `SELECT` on a table but forgetting `USAGE` on the database or schema.

### 13.3 Granting and Revoking

```sql
-- Privileges to roles
GRANT USAGE ON DATABASE sales TO ROLE analyst;
GRANT USAGE ON SCHEMA sales.curated TO ROLE analyst;
GRANT SELECT ON TABLE sales.curated.orders TO ROLE analyst;
GRANT SELECT ON ALL TABLES IN SCHEMA sales.curated TO ROLE analyst;      -- existing tables only
GRANT SELECT ON FUTURE TABLES IN SCHEMA sales.curated TO ROLE analyst;   -- tables created later
GRANT USAGE ON WAREHOUSE bi_wh TO ROLE analyst;

-- Roles to roles and users
GRANT ROLE analyst TO ROLE sysadmin;
GRANT ROLE analyst TO USER alice;

-- Delegation
GRANT SELECT ON TABLE sales.curated.orders TO ROLE lead WITH GRANT OPTION;

-- Revoke
REVOKE SELECT ON ALL TABLES IN SCHEMA sales.curated FROM ROLE analyst;
REVOKE ROLE analyst FROM USER alice;
REVOKE ROLE analyst FROM ROLE sysadmin;
```

### 13.4 Future Grants and "ALL" Grants

| Feature | Behaviour |
| --- | --- |
| `GRANT ... ON ALL <objects> IN SCHEMA/DATABASE` | Applies to **objects that exist now** only |
| `GRANT ... ON FUTURE <objects> IN SCHEMA/DATABASE` | Applies to objects **created afterwards** |
| **Both are usually needed** | `ALL` for what exists, `FUTURE` for what comes next |
| **Schema vs database future grants** | If a role has future grants of the same object type at both levels, the **schema-level** ones take precedence for that schema |
| **`CREATE OR REPLACE`** | The replaced object is new, so earlier explicit grants are lost; use `COPY CURRENT GRANTS` to carry them over |
| Inspect | `SHOW FUTURE GRANTS IN SCHEMA sales.curated;` |

### 13.5 Ownership

* One owner role per object; owner has all privileges and can grant them on.
* Transfer: `GRANT OWNERSHIP ON TABLE t TO ROLE r [REVOKE | COPY] CURRENT GRANTS;` Without the option, the transfer fails if outbound grants exist.
* Ownership transfer is the safe way to move administration, for example from `ACCOUNTADMIN` to `SYSADMIN` for objects created in the wrong role.

### 13.6 Managed Access Schemas

```sql
CREATE SCHEMA sales.curated WITH MANAGED ACCESS;
ALTER SCHEMA sales.curated ENABLE MANAGED ACCESS;
```

In a managed access schema, **object owners can no longer grant privileges on their objects**. Only the **schema owner** or a role with **`MANAGE GRANTS`** can. This centralizes grant control and prevents privilege sprawl from individual object owners.

<br></br>

## 14. Database Roles

A **database role** is a role scoped to one database. Name it `database.role`.

```sql
CREATE DATABASE ROLE sales.read;
GRANT USAGE ON SCHEMA sales.curated TO DATABASE ROLE sales.read;
GRANT SELECT ON ALL TABLES IN SCHEMA sales.curated TO DATABASE ROLE sales.read;

GRANT DATABASE ROLE sales.read TO ROLE analyst;        -- connect to an account role
GRANT DATABASE ROLE sales.read TO DATABASE ROLE sales.write;  -- hierarchy within the same database
SHOW GRANTS TO DATABASE ROLE sales.read;
```

| Property | Detail |
| --- | --- |
| **Granted to** | Account roles, or other database roles **in the same database** |
| **Cannot** | Be activated directly in a session (you activate an account role that holds it); own objects; have account roles granted to it |
| **Why use them** | Privileges travel with the database (clones, replication) and can be **shared** with consumers via secure data sharing |
| **Role hierarchy** | Database roles can inherit other database roles in the same database, giving read, write, admin tiers per database |

<br></br>

## 15. Designing a Role Hierarchy

### 15.1 The Recommended Pattern: Access Roles + Functional Roles

| Layer | Purpose | Example |
| --- | --- | --- |
| **Access roles** (object-level) | Hold privileges on a set of objects; named by object and level | `SALES_READ`, `SALES_WRITE`, `RAW_ETL_RW`, `BI_WH_USAGE` |
| **Functional roles** (business-level) | Represent a **job function**; inherit the access roles they need; granted to users | `ANALYST`, `DATA_ENGINEER`, `DATA_SCIENTIST` |
| **System roles** | Ownership and administration | `SYSADMIN`, `SECURITYADMIN`, `USERADMIN` |

```
 USERS ──► FUNCTIONAL ROLES ──► ACCESS ROLES ──► PRIVILEGES ON OBJECTS
 alice ──► ANALYST ───────────► SALES_READ ────► SELECT on sales.*
                    └─────────► BI_WH_USAGE ───► USAGE on bi_wh
 bob   ──► DATA_ENGINEER ─────► SALES_WRITE ───► INSERT/UPDATE on sales.*
                                    └─────────► (inherits SALES_READ)

 All functional roles ──► SYSADMIN ──► ACCOUNTADMIN
```

Benefits: privileges are defined once per object set; adding a person or a job function never requires touching object grants; auditing is simple.

### 15.2 End-to-End Example Script

```sql
-- 1. Create roles (USERADMIN owns them)
USE ROLE USERADMIN;
CREATE ROLE sales_read;       -- access roles
CREATE ROLE sales_write;
CREATE ROLE bi_wh_usage;
CREATE ROLE analyst;          -- functional roles
CREATE ROLE data_engineer;

-- 2. Create objects (SYSADMIN owns them), with managed access
USE ROLE SYSADMIN;
CREATE DATABASE sales;
CREATE SCHEMA sales.curated WITH MANAGED ACCESS;
CREATE WAREHOUSE bi_wh WAREHOUSE_SIZE = 'MEDIUM' AUTO_SUSPEND = 120 INITIALLY_SUSPENDED = TRUE;

-- 3. Grant privileges to ACCESS roles
GRANT USAGE ON DATABASE sales TO ROLE sales_read;
GRANT USAGE ON SCHEMA sales.curated TO ROLE sales_read;
GRANT SELECT ON ALL TABLES    IN SCHEMA sales.curated TO ROLE sales_read;
GRANT SELECT ON FUTURE TABLES IN SCHEMA sales.curated TO ROLE sales_read;

GRANT INSERT, UPDATE, DELETE, TRUNCATE ON ALL TABLES    IN SCHEMA sales.curated TO ROLE sales_write;
GRANT INSERT, UPDATE, DELETE, TRUNCATE ON FUTURE TABLES IN SCHEMA sales.curated TO ROLE sales_write;
GRANT CREATE TABLE ON SCHEMA sales.curated TO ROLE sales_write;

GRANT USAGE ON WAREHOUSE bi_wh TO ROLE bi_wh_usage;

-- 4. Build the hierarchy (access roles into functional roles, functional roles into SYSADMIN)
USE ROLE USERADMIN;
GRANT ROLE sales_read  TO ROLE sales_write;      -- write inherits read
GRANT ROLE sales_read  TO ROLE analyst;
GRANT ROLE bi_wh_usage TO ROLE analyst;
GRANT ROLE sales_write TO ROLE data_engineer;
GRANT ROLE bi_wh_usage TO ROLE data_engineer;
GRANT ROLE analyst       TO ROLE sysadmin;       -- roll custom roles up to SYSADMIN
GRANT ROLE data_engineer TO ROLE sysadmin;

-- 5. Assign to users
GRANT ROLE analyst TO USER alice;
ALTER USER alice SET DEFAULT_ROLE = analyst, DEFAULT_SECONDARY_ROLES = ('ALL');
```

### 15.3 Design Principles

| Principle | Detail |
| --- | --- |
| **Least privilege** | Grant only what a function needs; prefer read roles by default |
| **Separation of duties** | Different roles create users/roles (`USERADMIN`), grant access (`SECURITYADMIN`), and own data objects (`SYSADMIN`) |
| **Roll up to `SYSADMIN`** | Every custom role chain ends under `SYSADMIN` so objects stay administrable |
| **One owner role per domain** | Let a dedicated owner role (under `SYSADMIN`) own the objects in each database or domain, rather than whichever role happened to create them |
| **Environment separation** | Separate roles (or databases) per `DEV` / `TEST` / `PROD`, e.g., `PROD_SALES_READ` |
| **Naming conventions** | `<DOMAIN>_<LEVEL>` for access roles, job titles for functional roles, `SVC_<APP>` for service accounts |
| **Service accounts** | Dedicated roles and users per application, key-pair authentication, minimal privileges |
| **Central grant control** | Use managed access schemas so grants flow through one controlled path |
| **Use database roles** | For per-database access tiers, especially if databases are replicated or shared |

<br></br>

## 16. Inspecting and Auditing Roles and Grants

### 16.1 `SHOW` Commands

```sql
SHOW ROLES;                                  -- all roles
SHOW GRANTS TO ROLE analyst;                 -- privileges AND roles granted TO analyst (direct only)
SHOW GRANTS OF ROLE analyst;                 -- who/what the role is granted TO (users, parent roles)
SHOW GRANTS TO USER alice;                   -- roles granted to a user
SHOW GRANTS ON TABLE sales.curated.orders;   -- who has privileges ON the object
SHOW GRANTS ON ACCOUNT;                      -- account-level privileges
SHOW FUTURE GRANTS IN SCHEMA sales.curated;  -- future grant rules
```

**Important:** `SHOW GRANTS TO ROLE` shows **direct** grants only. Inherited privileges require walking the hierarchy.

### 16.2 `ACCOUNT_USAGE` Views

| View | Use |
| --- | --- |
| `GRANTS_TO_ROLES` | Every privilege and role grant to a role (role-to-role grants appear with `GRANTED_ON = 'ROLE'`) |
| `GRANTS_TO_USERS` | Roles granted to users |
| `ROLES` / `USERS` | Role and user inventory |
| `GRANTS_TO_ROLES` + `QUERY_HISTORY` | Joins for access reviews and "who could have run this" analysis |

`ACCOUNT_USAGE` has latency (up to a couple of hours) and includes deleted grants (`DELETED_ON` not null), so filter for current ones.

### 16.3 Walking the Role Hierarchy

```sql
-- All roles that SYSADMIN inherits (its children, at any depth)
WITH RECURSIVE role_tree (parent_role, child_role, depth) AS (
  SELECT grantee_name, name, 1
  FROM snowflake.account_usage.grants_to_roles
  WHERE granted_on = 'ROLE' AND granted_to = 'ROLE'
    AND privilege = 'USAGE' AND deleted_on IS NULL
    AND grantee_name = 'SYSADMIN'
  UNION ALL
  SELECT g.grantee_name, g.name, t.depth + 1
  FROM snowflake.account_usage.grants_to_roles g
  JOIN role_tree t ON g.grantee_name = t.child_role
  WHERE g.granted_on = 'ROLE' AND g.granted_to = 'ROLE'
    AND g.privilege = 'USAGE' AND g.deleted_on IS NULL
)
SELECT * FROM role_tree ORDER BY depth, parent_role, child_role;

-- Who holds ACCOUNTADMIN (should be a very short list)
SELECT grantee_name AS user_name
FROM snowflake.account_usage.grants_to_users
WHERE role = 'ACCOUNTADMIN' AND deleted_on IS NULL;
```

In these views, a role-to-role grant is stored as a `USAGE` privilege on the child role, with the **parent** as the grantee.

<br></br>

## 17. Best Practices and Pitfalls

**Roles**

* Limit `ACCOUNTADMIN` to a **small number of users** (at least two for continuity), with MFA, and never as a default role.
* Don't create objects with `ACCOUNTADMIN`; if it happens, transfer ownership to `SYSADMIN` or a domain owner role.
* Always roll custom roles up to `SYSADMIN`.
* Prefer **functional + access role** layering over granting object privileges directly to job roles.
* Use **future grants plus `ALL` grants** together; test with a newly created object.
* Use **managed access schemas** where governance matters.
* Avoid granting data privileges to `PUBLIC`.
* Review access regularly with `GRANTS_TO_ROLES` / `GRANTS_TO_USERS`; remove stale users and roles.
* Use **database roles** for shared or replicated databases.

**Common mistakes**

* Granting `SELECT` without `USAGE` on the database and schema.
* Granting roles in the **wrong direction** (parent and child reversed).
* Forgetting future grants, so new tables are invisible to the role.
* Using `CREATE OR REPLACE` and losing grants (use `COPY CURRENT GRANTS`).
* Expecting secondary roles to allow `CREATE` or to change object ownership.
* Leaving a user with `ACCOUNTADMIN` as the default role.
* Creating roles with `ACCOUNTADMIN` and leaving them outside the `SYSADMIN` branch.
* Treating `SHOW GRANTS TO ROLE` as the complete picture (it omits inherited privileges).
* Granting a role to a user and assuming it is active automatically (it is only if it is the default role, or the user switches or secondary roles are enabled).

<br></br>

## 18. How Parameters and Roles Interact

| Topic | Relationship |
| --- | --- |
| **No role-level parameters** | Parameters attach to account, user, session, or objects. To vary behaviour by job function, use user-level parameters or dedicated warehouses |
| **Privileges to change parameters** | Account: `ACCOUNTADMIN`; user: user owner (`USERADMIN` / `SECURITYADMIN`); object: owner or `MODIFY`; warehouse: `MODIFY` |
| **Default role is a user property, not a parameter** | `DEFAULT_ROLE` and `DEFAULT_SECONDARY_ROLES` are set with `ALTER USER` but have no hierarchy |
| **Session context** | The active role and the session parameters together define what a session can do and how it behaves |
| **Security parameters** | Account parameters like `REQUIRE_STORAGE_INTEGRATION_FOR_STAGE_CREATION` constrain what even privileged roles can do |
| **Policies follow a similar override model** | Network, password, authentication, and session policies are set at account and can be overridden at user level |

<br></br>

## 19. Quick Interview Q&A

| Question | Short Answer |
| --- | --- |
| What are the three parameter types? | Account (account only), Session (account, user, session), Object (account plus object levels such as database, schema, table, warehouse). |
| Parameter precedence for session parameters? | Session overrides User overrides Account overrides Snowflake default. |
| Parameter hierarchy for object parameters? | Account, Database, Schema, Table; the most specific explicit value wins, otherwise it inherits. |
| How do you see where a parameter value comes from? | `SHOW PARAMETERS ... IN <scope>` and read the `level` column. |
| What does `UNSET` do? | Removes the explicit value so the object inherits from the level above (or default). |
| What is `MIN_DATA_RETENTION_TIME_IN_DAYS`? | An account-only floor; effective retention is the greater of it and the object's own setting. |
| What if timeout is set on both warehouse and session? | The lowest non-zero value is enforced. |
| Parameter vs property? | Parameters have override levels; properties (e.g., `AUTO_SUSPEND`) are fixed attributes of one object. |
| Can parameters be set per role? | No. Use user-level parameters, dedicated warehouses, or resource monitors. |
| What are the system-defined roles? | `ORGADMIN`, `ACCOUNTADMIN`, `SECURITYADMIN`, `USERADMIN`, `SYSADMIN`, `PUBLIC`. |
| Who should own objects? | `SYSADMIN` or a custom owner role under it, not `ACCOUNTADMIN`. |
| What does `GRANT ROLE A TO ROLE B` mean? | B (parent) inherits A's (child's) privileges. |
| Can role A and role B be granted to each other? | No. Circular role grants are not allowed. |
| Why roll custom roles up to `SYSADMIN`? | So `SYSADMIN` (and `ACCOUNTADMIN`) can manage objects owned by those roles. |
| Primary vs secondary role? | Primary determines object ownership and `CREATE` authorization; secondary roles add their privileges to the session for access. |
| What is needed to query a table? | `USAGE` on database and schema, `SELECT` on the table, and `USAGE` on a warehouse. |
| Future grants vs `ALL` grants? | `ALL` covers existing objects; `FUTURE` covers objects created later. Use both. |
| What is a managed access schema? | A schema where only the schema owner or a `MANAGE GRANTS` role can grant privileges on its objects. |
| What is a database role? | A role scoped to one database; it can be granted to account roles, supports sharing, and cannot be activated directly or own objects. |
| Why use access roles and functional roles? | Define privileges once per object set (access), then compose them per job function (functional), which scales and simplifies audits. |
| How do you see inherited privileges? | Walk the hierarchy with `GRANTS_TO_ROLES` (recursive query) since `SHOW GRANTS TO ROLE` shows direct grants only. |
| What does `IS_ROLE_IN_SESSION` do? | Tells whether a role is active in the session, including through the hierarchy; used in masking and row access policies. |
| Who creates users and roles? | `USERADMIN` (create), `SECURITYADMIN` (manage grants globally). |
