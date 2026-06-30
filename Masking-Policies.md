Snowflake masking policies are **schema-level objects** that evaluate conditions *at query runtime*. They do not permanently alter data on disk (which would be static masking).

## The Dual-Silo Security Model

To maintain separation of duties, split your operations between a security admin (who creates policies) and an object owner/data steward (who applies them).

```sql
-- 1. Create a dedicated Masking Administrator Role
USE ROLE USERADMIN;
CREATE ROLE MASKING_ADMIN;

-- 2. Grant Policy-Creation and Global-Apply Privileges
USE ROLE SECURITYADMIN;
GRANT CREATE MASKING_ADMIN ON SCHEMA my_db.security_schema TO ROLE MASKING_ADMIN;
GRANT APPLY MASKING POLICY ON ACCOUNT TO ROLE MASKING_ADMIN;

-- 3. (Optional) Decentralize Application to Table Owners
-- This allows the table owner role to apply or unset this specific policy
GRANT APPLY ON MASKING POLICY security_schema.email_mask TO ROLE TABLE_OWNER;

```

## 2. Policy Definitions

Every policy accepts an input argument matching the column's data type and **must** return that exact same data type.

### Common Masking Templates

```sql
USE ROLE MASKING_ADMIN;
USE SCHEMA my_db.security_schema;

-- Template A: Full Masking (String)
CREATE OR REPLACE MASKING_POLICY string_full_mask AS (val STRING) 
  RETURNS STRING ->
    CASE 
      WHEN CURRENT_ROLE() IN ('SECURITY_ADMIN', 'HR_MGR') THEN val 
      ELSE '*********' 
    END;

-- Template B: Partial Masking (Email Domain Visibility Only)
CREATE OR REPLACE MASKING_POLICY email_partial_mask AS (val STRING) 
  RETURNS STRING ->
    CASE 
      WHEN CURRENT_ROLE() = 'ANALYST' THEN REGEXP_REPLACE(val, '^.*@', 'masked_user@')
      ELSE val -- Default allows read if role isn't restricted, or reverse it based on security posture
    END;

-- Template C: Numeric / Date Masking (Must preserve Data Type!)
CREATE OR REPLACE MASKING_POLICY salary_mask AS (val NUMBER) 
  RETURNS NUMBER ->
    CASE 
      WHEN IS_ROLE_IN_SESSION('FINANCE') THEN val 
      ELSE -999999 -- Use a distinct indicator value since you cannot return a string text like 'MASKED'
    END;

```

### Context Verification Functions

Use these context functions inside your `CASE WHEN` logic to route permissions:

* `CURRENT_ROLE()`: Returns the primary active role of the current session.
* `IS_ROLE_IN_SESSION('ROLE_NAME')`: **Best Practice.** Checks if the role is active anywhere in the user's current active role hierarchy (handles secondary roles cleanly).
* `INVOKER_ROLE()`: Evaluates the role executing the statement (critical when checking access through secure views).

## 3. Dynamic Masking vs. Conditional Masking

You can build logic based solely on the target column, or evaluate a secondary column in the same row to determine visibility (Conditional Masking).

| Metric | Simple Dynamic Masking | Conditional Masking |
| --- | --- | --- |
| **Arguments** | Accepts exactly 1 argument (the target column value). | Accepts multiple arguments (target column + conditional column columns). |
| **Scope** | Evaluates permissions strictly based on role/context. | Evaluates role *and* structural data attributes (e.g., region, visibility flags). |

### Conditional Masking Example

```sql
-- The first argument is ALWAYS the column being masked. Additional columns are parameters.
CREATE OR REPLACE MASKING_POLICY conditional_email_mask 
  AS (email STRING, visibility_flag STRING) 
  RETURNS STRING ->
    CASE 
      WHEN CURRENT_ROLE() = 'ADMIN' THEN email
      WHEN visibility_flag = 'PUBLIC' THEN email
      ELSE '***HIDDEN***'
    END;

```
## 4. Applying and Unsetting Policies

Policies can be bound during table creation or dynamically attached to active tables and views.

### On New Objects

```sql
-- Standard Application
CREATE TABLE customers (
  id INT,
  email STRING MASKING POLICY email_partial_mask
);

-- Conditional Application (Explicitly mapping conditional arguments via USING)
CREATE TABLE localized_users (
  email STRING MASKING POLICY conditional_email_mask USING (email, region_visibility),
  region_visibility STRING
);

```

### On Existing Objects

```sql
-- Attach a policy
ALTER TABLE customers MODIFY COLUMN email SET MASKING POLICY email_partial_mask;

-- Attach a conditional policy
ALTER TABLE localized_users MODIFY COLUMN email SET MASKING POLICY conditional_email_mask USING (email, region_visibility);

-- Remove a policy (Required before you can drop or replace the policy object!)
ALTER TABLE customers MODIFY COLUMN email UNSET MASKING POLICY;

```
## 5. Scaled Administration: Tag-Based Masking

Instead of manually mapping columns across thousands of tables, assign a policy directly to an **Object Tag**. Any column assigned that tag automatically inherits the policy.

```sql
USE ROLE MASKING_ADMIN;

-- 1. Create the governance tag
CREATE TAG governance.pii_type;

-- 2. Bind the masking policy to the tag for a specific data type
ALTER TAG governance.pii_type SET MASKING POLICY security_schema.string_full_mask;

-- 3. Simply tag your asset columns. The policy is instantly active.
ALTER TABLE hr.employees MODIFY COLUMN ssn SET TAG governance.pii_type = 'SSN';

```

## 6. Crucial Guardrails & Troubleshooting

⚠️ **Query Rewrite Overhead:** Snowflake injects masking policy expressions inline into query execution trees. This rewrite executes wherever the column is evaluated—including `PROJECTIONS`, `JOIN` predicates, `WHERE` clauses, and `GROUP BY` statements. Minimize massive subqueries or complex UDFs inside your policy body to protect query execution times.

### Common Errors & Fixes

* **Error:** `Policy cannot be dropped/replaced as it is associated with one or more entities.`
* **Fix:** You cannot drop an active policy. Query dependencies first, run `ALTER TABLE ... MODIFY COLUMN ... UNSET MASKING POLICY` on all targets, and then drop it.

* **Error:** `Unsupported feature CREATE ON MASKING POLICY COLUMN.`
* **Fix:** Ensure you are not attempting to apply masking policies to unsupported features or incompatible virtual column configurations. When dealing with external tables, remember that the virtual column can override policies inherited from the primary `VALUE` blob if `EXEMPT_OTHER_POLICIES = TRUE` is toggled on the base policy.


### Auditing Policies

```sql
-- Find every asset using a specific policy
SELECT * FROM TABLE(information_schema.policy_references(policy_name => 'security_schema.email_mask'));

-- View all masking definitions inside the account usage metadata
SELECT * FROM snowflake.account_usage.masking_policies WHERE deleted IS NULL;

```
