# Data Sharing — Security, Governance & Best Practices

> Snowflake Edition requirements noted inline. All SQL uses uppercase keywords, `snake_case` identifiers.

---

## 2.1 RBAC Hierarchy (Least Privilege)

### Provider-Side Roles

```
ACCOUNTADMIN
    └── SYSADMIN
          ├── SHARE_ADMIN          (CREATE SHARE, MANAGE SHARE TARGET)
          ├── LISTING_ADMIN        (CREATE DATA EXCHANGE LISTING)
          └── SHARE_MONITOR        (read-only via ACCOUNT_USAGE)
```

```sql
-- Provider: create custom roles
USE ROLE SECURITYADMIN;

CREATE ROLE IF NOT EXISTS SHARE_ADMIN
  COMMENT = 'Can create and manage outbound shares';
CREATE ROLE IF NOT EXISTS LISTING_ADMIN
  COMMENT = 'Can create and manage Marketplace listings';
CREATE ROLE IF NOT EXISTS SHARE_MONITOR
  COMMENT = 'Read-only monitoring of share usage';

-- Wire into hierarchy
GRANT ROLE SHARE_ADMIN   TO ROLE SYSADMIN;
GRANT ROLE LISTING_ADMIN TO ROLE SYSADMIN;
GRANT ROLE SHARE_MONITOR TO ROLE SYSADMIN;

-- Provider privileges
USE ROLE ACCOUNTADMIN;

GRANT CREATE SHARE ON ACCOUNT TO ROLE SHARE_ADMIN;
GRANT MANAGE SHARE TARGET ON ACCOUNT TO ROLE SHARE_ADMIN;

GRANT CREATE DATA EXCHANGE LISTING ON ACCOUNT TO ROLE LISTING_ADMIN;

-- SHARE_MONITOR: read-only via ACCOUNT_USAGE views
GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE SHARE_MONITOR;

-- Grant SHARE_ADMIN access to source objects
GRANT USAGE ON DATABASE ANALYTICS_DB TO ROLE SHARE_ADMIN;
GRANT USAGE ON SCHEMA ANALYTICS_DB.SHARED TO ROLE SHARE_ADMIN;
GRANT SELECT ON ALL TABLES IN SCHEMA ANALYTICS_DB.SHARED TO ROLE SHARE_ADMIN;
GRANT SELECT ON ALL VIEWS  IN SCHEMA ANALYTICS_DB.SHARED TO ROLE SHARE_ADMIN;
```

### Consumer-Side Roles

```
ACCOUNTADMIN
    └── SYSADMIN
          └── SHARE_IMPORTER       (IMPORT SHARE, CREATE DATABASE)
                └── SHARED_DATA_READER  (SELECT on shared objects)
```

```sql
-- Consumer: create custom roles
USE ROLE SECURITYADMIN;

CREATE ROLE IF NOT EXISTS SHARE_IMPORTER
  COMMENT = 'Can import inbound shares and create databases from them';
CREATE ROLE IF NOT EXISTS SHARED_DATA_READER
  COMMENT = 'Read-only access to shared data';

GRANT ROLE SHARED_DATA_READER TO ROLE SHARE_IMPORTER;
GRANT ROLE SHARE_IMPORTER     TO ROLE SYSADMIN;

-- Consumer privileges
USE ROLE ACCOUNTADMIN;

GRANT IMPORT SHARE   ON ACCOUNT TO ROLE SHARE_IMPORTER;
GRANT CREATE DATABASE ON ACCOUNT TO ROLE SHARE_IMPORTER;

-- After creating DB from share:
-- GRANT IMPORTED PRIVILEGES ON DATABASE <shared_db> TO ROLE SHARED_DATA_READER;
```

> **Note:** `CREATE SHARE` and `MANAGE SHARE TARGET` were split from the legacy `CREATE SHARE` global privilege (BCR 2024_07). Existing grants were automatically migrated.

---

## 2.2 Secure Views & Secure UDFs

### Why Mandatory

Shares enforce `SECURE_OBJECTS_ONLY = TRUE` by default. Only **secure views**, **secure UDFs**, and **secure UDTFs** can be added to a share. This prevents consumers from reverse-engineering underlying table structures or data through query plans.

A secure view:
- Hides the view definition from unauthorized users.
- Bypasses the query optimizer's push-down of predicates from the outer query into the view, preventing data leakage through side channels.

### Per-Consumer Filtering with CURRENT_ACCOUNT()

Use `CURRENT_ACCOUNT()` inside a secure view to return different rows to different consumer accounts.

```sql
CREATE OR REPLACE SECURE VIEW ANALYTICS_DB.SHARED.V_ORDERS
AS
SELECT
    ORDER_ID,
    ORDER_DATE,
    CUSTOMER_ID,
    AMOUNT,
    REGION
FROM ANALYTICS_DB.INTERNAL.ORDERS
WHERE ACCOUNT_LOCATOR = CURRENT_ACCOUNT();
```

### Testing with SIMULATED_DATA_SHARING_CONSUMER

Before publishing, simulate the consumer experience on the provider side:

```sql
ALTER SESSION SET SIMULATED_DATA_SHARING_CONSUMER = '<consumer_account_locator>';

-- Test the secure view as if you were the consumer
SELECT * FROM ANALYTICS_DB.SHARED.V_ORDERS LIMIT 10;

-- Verify that CURRENT_ACCOUNT() returns the simulated value
SELECT CURRENT_ACCOUNT();

-- Reset
ALTER SESSION UNSET SIMULATED_DATA_SHARING_CONSUMER;
```

> **Tip:** Combine with `EXPLAIN` to verify no predicate push-down leaks filter logic.

---

## 2.3 Database Roles in Shares

Database roles enable **granular segmentation** within a single share. Instead of granting object-level privileges directly to the share, you grant a database role to the share. The consumer then maps that database role to a local account role.

### Provider Side

```sql
USE ROLE SHARE_ADMIN;

-- 1. Create database roles for segmentation
CREATE DATABASE ROLE IF NOT EXISTS ANALYTICS_DB.DR_FINANCE;
CREATE DATABASE ROLE IF NOT EXISTS ANALYTICS_DB.DR_MARKETING;

-- 2. Grant objects to each database role
GRANT USAGE ON SCHEMA ANALYTICS_DB.SHARED TO DATABASE ROLE ANALYTICS_DB.DR_FINANCE;
GRANT SELECT ON VIEW ANALYTICS_DB.SHARED.V_REVENUE TO DATABASE ROLE ANALYTICS_DB.DR_FINANCE;

GRANT USAGE ON SCHEMA ANALYTICS_DB.SHARED TO DATABASE ROLE ANALYTICS_DB.DR_MARKETING;
GRANT SELECT ON VIEW ANALYTICS_DB.SHARED.V_CAMPAIGNS TO DATABASE ROLE ANALYTICS_DB.DR_MARKETING;

-- 3. Grant database roles to the share
GRANT DATABASE ROLE ANALYTICS_DB.DR_FINANCE   TO SHARE ANALYTICS_SHARE;
GRANT DATABASE ROLE ANALYTICS_DB.DR_MARKETING TO SHARE ANALYTICS_SHARE;
```

### Consumer Side

```sql
USE ROLE SHARE_IMPORTER;

-- Create DB from share (if not already done)
CREATE DATABASE IF NOT EXISTS PROVIDER_ANALYTICS FROM SHARE <provider_account>.ANALYTICS_SHARE;

-- Map shared database roles to local account roles
USE ROLE SECURITYADMIN;
GRANT DATABASE ROLE PROVIDER_ANALYTICS.DR_FINANCE TO ROLE FINANCE_ANALYSTS;
GRANT DATABASE ROLE PROVIDER_ANALYTICS.DR_MARKETING TO ROLE MARKETING_TEAM;
```

> **Limitations:** `FUTURE` grants on objects are **not** supported for database roles that are granted to shares. You must grant privileges explicitly on each new object.

---

## 2.4 Row Access Policies

> **Requires:** Enterprise Edition or higher.

Row access policies (RAP) let the provider control which rows each consumer can see. In a data sharing context, `CURRENT_ROLE()` returns `NULL` for the consumer — you **must** use `IS_DATABASE_ROLE_IN_SESSION()` instead.

### Key Constraints

- The mapping table **must** reside in the **same database** as the protected table.
- The policy body must reference only objects in the same database.

### SQL Example

```sql
-- Mapping table: which database role sees which regions
CREATE TABLE ANALYTICS_DB.SHARED.REGION_ACCESS_MAP (
    DB_ROLE_NAME VARCHAR,
    ALLOWED_REGION VARCHAR
);

INSERT INTO ANALYTICS_DB.SHARED.REGION_ACCESS_MAP VALUES
    ('DR_FINANCE',   'NA'),
    ('DR_FINANCE',   'EMEA'),
    ('DR_MARKETING', 'NA');

-- Row access policy
CREATE OR REPLACE ROW ACCESS POLICY ANALYTICS_DB.SHARED.RAP_REGION
AS (region_val VARCHAR) RETURNS BOOLEAN ->
    EXISTS (
        SELECT 1
        FROM ANALYTICS_DB.SHARED.REGION_ACCESS_MAP m
        WHERE IS_DATABASE_ROLE_IN_SESSION(m.DB_ROLE_NAME)
          AND m.ALLOWED_REGION = region_val
    );

-- Attach to the table or secure view
ALTER VIEW ANALYTICS_DB.SHARED.V_ORDERS
  ADD ROW ACCESS POLICY ANALYTICS_DB.SHARED.RAP_REGION ON (REGION);
```

> **Warning:** If you use `CURRENT_ROLE()` instead of `IS_DATABASE_ROLE_IN_SESSION()`, consumers will see **zero rows** because `CURRENT_ROLE()` evaluates to `NULL` in the consumer context.

---

## 2.5 Masking Policies

> **Requires:** Enterprise Edition or higher.

Masking policies follow the same `IS_DATABASE_ROLE_IN_SESSION()` pattern as row access policies. They control **column-level** visibility per consumer.

### Key Constraints

- External tokenisation functions are **not** supported with shares (the consumer cannot call provider-side external functions).
- Tag-based masking policies are enforced on shared data; however, tags and their policy assignments must be set by the provider.

### SQL Example

```sql
CREATE OR REPLACE MASKING POLICY ANALYTICS_DB.SHARED.MASK_EMAIL
AS (val VARCHAR) RETURNS VARCHAR ->
    CASE
        WHEN IS_DATABASE_ROLE_IN_SESSION('DR_FINANCE')
            THEN val
        ELSE REGEXP_REPLACE(val, '.+@', '****@')
    END;

ALTER TABLE ANALYTICS_DB.INTERNAL.CUSTOMERS
  MODIFY COLUMN EMAIL
  SET MASKING POLICY ANALYTICS_DB.SHARED.MASK_EMAIL;
```

### Tag-Based Masking

```sql
CREATE OR REPLACE TAG ANALYTICS_DB.SHARED.PII_LEVEL
  ALLOWED_VALUES 'HIGH', 'MEDIUM', 'LOW';

ALTER TAG ANALYTICS_DB.SHARED.PII_LEVEL
  SET MASKING POLICY ANALYTICS_DB.SHARED.MASK_EMAIL;

-- Apply tag to columns
ALTER TABLE ANALYTICS_DB.INTERNAL.CUSTOMERS
  MODIFY COLUMN EMAIL
  SET TAG ANALYTICS_DB.SHARED.PII_LEVEL = 'HIGH';
```

---

## 2.6 Network Security & Private Connectivity

> **Requires:** Business Critical Edition or higher.

### AWS PrivateLink / Azure Private Link / GCP Private Service Connect

Private connectivity ensures share traffic never traverses the public internet.

```sql
-- Provider: authorize the consumer's AWS account for PrivateLink
SELECT SYSTEM$AUTHORIZE_PRIVATELINK(
    '<consumer_aws_account_id>',
    '<consumer_vpc_endpoint_id>'
);

-- Retrieve PrivateLink configuration
SELECT SYSTEM$GET_PRIVATELINK_CONFIG();
```

### Restrict Share Access to PrivateLink Only

```sql
-- Enforce PrivateLink-only access at the account level (Business Critical+)
SELECT SYSTEM$ENFORCE_PRIVATELINK_ACCESS_ONLY();
```

### Network Policies for Shares

Network policies can be applied at the account level to restrict IP ranges that can access shared data:

```sql
CREATE NETWORK POLICY SHARE_ACCESS_POLICY
  ALLOWED_IP_LIST = ('10.0.0.0/8', '172.16.0.0/12')
  COMMENT = 'Restrict share access to corporate VPN';

ALTER ACCOUNT SET NETWORK_POLICY = SHARE_ACCESS_POLICY;
```

---

## 2.7 Cross-Region & Cross-Cloud

### Auto-Fulfillment

Auto-fulfillment replicates shared data to consumer regions automatically. The provider enables it per listing, and Snowflake manages the replication.

```sql
-- Provider: trigger an on-demand refresh for a listing
SELECT SYSTEM$TRIGGER_LISTING_REFRESH('<listing_global_name>');
```

### Refresh Modes

| Mode | Description |
|------|-------------|
| `TRIGGER` | Manual refresh via `SYSTEM$TRIGGER_LISTING_REFRESH` |
| `INTERVAL` | Fixed interval (e.g., `10 MINUTE`) |
| `CRON` | Cron-based schedule (e.g., `USING CRON 0 */6 * * * UTC`) |

### SUB_DATABASE Mode

For large databases, replicate only specific schemas/objects to reduce cross-region replication costs:

```sql
ALTER LISTING <listing_name> SET
  AUTO_FULFILLMENT_REPLICATION_OPTION = 'SUB_DATABASE';
```

### REFERENCE_USAGE for Cross-Database Views

When a secure view references tables in a different database, the share needs `REFERENCE_USAGE` on that database:

```sql
GRANT REFERENCE_USAGE ON DATABASE LOOKUP_DB TO SHARE ANALYTICS_SHARE;
```

---

## 2.8 Audit & Monitoring

### DATA_SHARING_USAGE Views

```sql
-- Listing access history (provider sees consumer access)
SELECT *
FROM SNOWFLAKE.DATA_SHARING_USAGE.LISTING_ACCESS_HISTORY
WHERE LISTING_NAME = 'MY_LISTING'
ORDER BY ACCESS_DATE DESC
LIMIT 100;
```

### ACCOUNT_USAGE.ACCESS_HISTORY

> **Requires:** Enterprise Edition or higher.

```sql
-- Query-level access to shared objects
SELECT
    QUERY_ID,
    USER_NAME,
    DIRECT_OBJECTS_ACCESSED,
    BASE_OBJECTS_ACCESSED,
    QUERY_START_TIME
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
WHERE ARRAY_SIZE(BASE_OBJECTS_ACCESSED) > 0
  AND BASE_OBJECTS_ACCESSED[0]:objectDomain::STRING = 'Table'
ORDER BY QUERY_START_TIME DESC
LIMIT 50;
```

### LISTING_TELEMETRY_DAILY

```sql
-- Daily telemetry for Marketplace listings
SELECT
    EVENT_DATE,
    LISTING_NAME,
    LISTING_DISPLAY_NAME,
    EVENT_TYPE,
    SNOWFLAKE_REGION,
    CONSUMER_ACCOUNTS_DAILY
FROM SNOWFLAKE.DATA_SHARING_USAGE.LISTING_TELEMETRY_DAILY
WHERE EVENT_DATE >= DATEADD('day', -30, CURRENT_DATE())
ORDER BY EVENT_DATE DESC;
```

### REPLICATION_USAGE_HISTORY

```sql
-- Cross-region replication costs
SELECT
    START_TIME,
    END_TIME,
    DATABASE_NAME,
    CREDITS_USED,
    BYTES_TRANSFERRED
FROM SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_USAGE_HISTORY
WHERE START_TIME >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY START_TIME DESC;
```

### Alert: Inactive Consumers

```sql
CREATE OR REPLACE ALERT ANALYTICS_DB.SHARED.ALERT_INACTIVE_CONSUMERS
  WAREHOUSE = ADMIN_WH
  SCHEDULE  = 'USING CRON 0 8 * * MON UTC'
  IF (EXISTS (
      SELECT 1
      FROM SNOWFLAKE.DATA_SHARING_USAGE.LISTING_ACCESS_HISTORY
      WHERE LISTING_NAME = 'MY_LISTING'
      GROUP BY CONSUMER_ACCOUNT_LOCATOR
      HAVING MAX(ACCESS_DATE) < DATEADD('day', -30, CURRENT_DATE())
  ))
  THEN
    CALL SYSTEM$SEND_EMAIL(
        'SHARE_ALERTS',
        'data-ops@company.com',
        'Inactive Share Consumers',
        'One or more consumers have not accessed the listing in 30+ days.'
    );

ALTER ALERT ANALYTICS_DB.SHARED.ALERT_INACTIVE_CONSUMERS RESUME;
```

---

## 2.9 Naming Conventions

| Object | Pattern | Example |
|--------|---------|---------|
| Outbound Share | `<DOMAIN>_SHARE` | `ANALYTICS_SHARE` |
| Consumer DB (from share) | `<PROVIDER>_<DOMAIN>` | `ACME_ANALYTICS` |
| Shared Schema | `SHARED` or `<AUDIENCE>_SHARED` | `ANALYTICS_DB.SHARED` |
| Database Role (share) | `DR_<SEGMENT>` | `DR_FINANCE` |
| Provider Role | `SHARE_ADMIN`, `LISTING_ADMIN`, `SHARE_MONITOR` | `SHARE_ADMIN` |
| Consumer Role | `SHARE_IMPORTER`, `SHARED_DATA_READER` | `SHARE_IMPORTER` |
| Secure View | `V_<ENTITY>` | `V_ORDERS` |
| Row Access Policy | `RAP_<DIMENSION>` | `RAP_REGION` |
| Masking Policy | `MASK_<COLUMN_TYPE>` | `MASK_EMAIL` |
| Tag | `<CLASSIFICATION>_LEVEL` | `PII_LEVEL` |
| Listing | `<DOMAIN>_LISTING_<AUDIENCE>` | `ANALYTICS_LISTING_EXTERNAL` |
| Network Policy | `<PURPOSE>_POLICY` | `SHARE_ACCESS_POLICY` |

---

## 2.10 Cost Model Summary

| Component | Provider Pays | Consumer Pays |
|-----------|:------------:|:------------:|
| Storage of shared data | ✅ | — |
| Compute for consumer queries | — | ✅ |
| Cross-region replication (auto-fulfillment) | ✅ | — |
| Secure view overhead (optimizer bypass) | — | ✅ (marginal) |
| Reader account compute | ✅ | — (reader has no billing) |
| Listing telemetry / metadata | ✅ (included) | — |
| PrivateLink data transfer | ✅ + ✅ (both sides pay respective cloud provider) | ✅ |
| Row/masking policy evaluation | — | ✅ (runs on consumer warehouse) |
| Replication for disaster recovery | ✅ | — |
| Snowflake Marketplace fee | — | — (no Snowflake fee; cloud egress may apply) |
