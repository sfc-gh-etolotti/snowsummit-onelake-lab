# Building an Iceberg Lakehouse with Snowflake and Microsoft OneLake

## Overview

Data architectures are consolidating around open table formats and flexible compute engines. As data volumes grow and AI workloads become more demanding, maintaining multiple copies of the same datasets across systems is no longer practical. The combination of OneLake storage, Apache Iceberg tables, and Snowflake's execution engine provides a clean, sustainable model that meets the needs of this architecture.

Apache Iceberg serves as the shared table layer between Snowflake and Microsoft Fabric. Iceberg's metadata structure, versioning, and concurrency model allow multiple engines to read and write the same tables without conflict. This removes the need for synchronization pipelines, custom connectors, or duplicated storage zones. The data lives once in OneLake, and Snowflake can operate on it directly through the Iceberg REST catalog.

This lab demonstrates the full bi-directional workflow: uploading raw retail data to a Fabric Lakehouse, reading it zero-copy in Snowflake through a Catalog Linked Database, transforming it into a star schema, and writing the curated results back to OneLake as Snowflake-managed Iceberg tables -- instantly available in Power BI and other Fabric services with no Snowflake connector required.

### Prerequisites

- A [Snowflake account](https://signup.snowflake.com/) with ACCOUNTADMIN access (trial works)
- A [Microsoft Fabric workspace](https://learn.microsoft.com/en-us/fabric/get-started/fabric-trial) with capacity (trial works)
- An Azure subscription for App Registration (comes with Fabric trial)
- Both platforms should have public internet access (no VNet)

### What You Will Learn

- Creating a Catalog Integration and External Volume to connect Snowflake to OneLake Iceberg tables
- Setting up a Catalog Linked Database (CLD) for zero-copy reads from OneLake into Snowflake
- Creating Snowflake-managed Iceberg tables that write Parquet files directly to OneLake
- Connecting a Snowflake database to OneLake using the Snowsight wizard
- Transforming raw data into a star schema directly as Iceberg tables

### What You Will Build

```
Fabric Lakehouse (RetailRaw)
  4 raw CSV tables (transactions, products, stores, customers)
     |
     v  Catalog Linked Database (zero-copy read via Iceberg REST Catalog)
Snowflake
  Reads raw data directly from OneLake Parquet files -- no data copied
     |
     v  Star schema transformation (Snowflake compute)
Snowflake-managed Iceberg (SNOWSUMMIT_ANALYTICS_ONELAKE)
  5 curated tables written as Parquet to OneLake, managed by Snowflake Horizon
     |
     v  Native read (Direct Lake mode)
Fabric / Power BI
  Reads the Iceberg tables directly from OneLake -- no Snowflake connector needed
```

---

## How to Run This Lab

This lab is designed to be delivered through **[Cortex Code (CoCo)](https://docs.snowflake.com/en/user-guide/ui-snowsight/cortex-code)** -- Snowflake's built-in AI IDE. CoCo explains every concept before executing SQL, pauses for your acknowledgement at each step, and guides you through the manual Fabric/Azure configuration steps. Nothing executes without a "go".

### Step 1: Clone or Download This Repo

```bash
git clone https://github.com/sfc-gh-etolotti/snowsummit-onelake-lab.git
```

Or download as a ZIP from the green **Code** button above.

### Step 2: Upload the Skill to CoCo

1. Open **Snowsight** and navigate to **Cortex Code (CoCo)**
2. Click the **+** button -> **Upload Skill**
3. Select the `SnowSummit_Skills` folder from this repo
4. CoCo will register the skill automatically

### Step 3: Start the Lab

Type **"snowsummit"** in CoCo and say **"go"** to begin. CoCo will walk you through each phase:

| Phase | What Happens | Est. Time |
|-------|-------------|-----------|
| 0 | Fabric workspace setup, Lakehouse creation, CSV upload, App Registration | 20 min |
| 1 | Collect 5 Azure/Fabric values | 5 min |
| 2 | Snowflake infrastructure (Catalog Integration, External Volume, consent, SP grants) | 15 min |
| 3 | Catalog Linked Database + zero-copy query validation | 10 min |
| 4 | OneLake write-back setup (database, Fabric connection, Snowsight wizard) | 15 min |
| 5 | Star schema as Iceberg tables directly in OneLake | 10 min |
| 6 | Recap + what's next | 5 min |
| 7 | Cleanup | 5 min |

**Total estimated time: ~90 minutes**

---

## Manual Walkthrough

If you prefer to follow along manually in a Snowflake SQL worksheet, all steps and queries are documented below.

## Step 1: Prepare the Fabric Environment

### 1a. Create a Fabric Workspace

1. Go to [app.fabric.microsoft.com](https://app.fabric.microsoft.com)
2. Sign in with your Microsoft account (work or school account required)
3. Click **Workspaces** -> **+ New workspace**
4. Name it `SnowSummitLab`, expand **Advanced** -> select **Trial** license mode -> **Apply**

### 1b. Enable Tenant Admin Settings

Two settings must be enabled in the **Fabric Admin Portal** :

1. Open **Settings** (gear icon) -> **Admin portal**
2. Under **Tenant settings** -> **OneLake settings**: Enable **"Users can access data stored in OneLake with apps external to Fabric"**
3. Under **Tenant settings** -> **Developer settings**: Enable **"Service principals can use Fabric APIs"**

### 1c. Create the RetailRaw Lakehouse

1. In your workspace, click **+ New item** -> **Lakehouse**
2. Name it exactly: **`RetailRaw`**
3. Click **Create**

### 1d. Upload and Load CSV Files

Upload the 4 CSV files from the `DataFiles/` folder in this repo:

| File | Description | ~Rows |
|------|-------------|-------|
| `raw_transactions.csv` | Retail transactions, 2024 | ~5,000 |
| `product_catalog.csv` | Product master with cost and category | ~250 |
| `store_locations.csv` | 55 store locations across 5 regions | 55 |
| `customer_profiles.csv` | Loyalty program customers | 500 |

1. In your Lakehouse, click **Get data** -> **Upload files**
2. Upload all 4 CSVs to the **Files** section
3. For each file: right-click -> **Load to Tables** -> **Load**

After loading, verify 4 tables appear under the **Tables** section.

### 1e. Copy Your Lakehouse URL

Copy the full browser URL from the Lakehouse Explorer page. Format:
```
https://app.fabric.microsoft.com/groups/<workspace-guid>/lakehouses/<lakehouse-guid>
```

---

## Step 2: Create the Azure App Registration

1. Go to [portal.azure.com](https://portal.azure.com) -> search **"App registrations"** -> **+ New registration**
2. Name: `SnowSummitLab_OAuth_Client`, Single tenant, no redirect URI -> **Register**
3. Under **API permissions** -> **+ Add a permission** -> **Azure Storage** -> **Delegated** -> check `user_impersonation` -> **Add** -> **Grant admin consent**
4. Under **Certificates & secrets** -> **+ New client secret** -> copy the **Value** immediately (shown only once)
5. Back on **Overview**, click **"Managed application in local directory"** link -> note the display name

You now have 5 values:
- **Azure Tenant ID** (Overview -> Directory (tenant) ID)
- **OAuth App Client ID** (Overview -> Application (client) ID)
- **OAuth Client Secret** (the Value you just copied)
- **Managed App Name** (the Enterprise App display name)
- **Fabric Lakehouse URL** (copied in Step 1e)

---

## Step 3: Configure Snowflake Variables

Open a new SQL worksheet in Snowsight and run:

```sql
USE ROLE ACCOUNTADMIN;

-- Replace these with your actual values
SET azure_tenant_id                  = '<your-tenant-id>';
SET azure_oauth_app_client_id        = '<your-client-id>';
SET azure_oauth_client_secret_value  = '<your-client-secret>';
SET azure_oauth_app_managed_app_name = '<your-managed-app-name>';
SET fabric_lakehouse_url             = '<your-lakehouse-url>';
SET fabric_lakehouse_name            = 'RetailRaw';

-- Derived variables (do not edit)
SET snowflake_catalog_integration_name = (SELECT 'SnowSummit_' || $fabric_lakehouse_name || '_IRC_INT');
SET snowflake_external_volume_name     = (SELECT 'SnowSummit_' || $fabric_lakehouse_name || '_EXTERNAL_VOLUME');
SET snowflake_cld_name                 = (SELECT 'SnowSummit_' || $fabric_lakehouse_name || '_CLDB');

SET fabric_workspace_id = (
    SELECT REGEXP_SUBSTR($fabric_lakehouse_url,
        'groups/([0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})', 1, 1, 'e', 1)
);
SET fabric_data_item_id = (
    SELECT REGEXP_SUBSTR($fabric_lakehouse_url,
        'lakehouses/([0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})', 1, 1, 'e', 1)
);
SET catalog_name     = (SELECT CONCAT_WS('/', $fabric_workspace_id, $fabric_data_item_id));
SET oauth_token_uri  = (SELECT CONCAT('https://login.microsoftonline.com/', $azure_tenant_id, '/oauth2/v2.0/token'));
SET storage_base_url = (SELECT CONCAT('azure://onelake.dfs.fabric.microsoft.com/', $catalog_name));

-- Validate all variables
WITH validate AS (
    SELECT 'azure_tenant_id' AS variable_name, $azure_tenant_id AS value UNION ALL
    SELECT 'azure_oauth_app_client_id', $azure_oauth_app_client_id UNION ALL
    SELECT 'azure_oauth_client_secret_value', LEFT($azure_oauth_client_secret_value, 4) || '****' UNION ALL
    SELECT 'azure_oauth_app_managed_app_name', $azure_oauth_app_managed_app_name UNION ALL
    SELECT 'fabric_workspace_id', $fabric_workspace_id UNION ALL
    SELECT 'fabric_data_item_id', $fabric_data_item_id UNION ALL
    SELECT 'catalog_name', $catalog_name UNION ALL
    SELECT 'oauth_token_uri', $oauth_token_uri UNION ALL
    SELECT 'storage_base_url', $storage_base_url UNION ALL
    SELECT 'snowflake_catalog_integration_name', $snowflake_catalog_integration_name UNION ALL
    SELECT 'snowflake_external_volume_name', $snowflake_external_volume_name UNION ALL
    SELECT 'snowflake_cld_name', $snowflake_cld_name
)
SELECT * FROM validate;
```

**Expected**: 12 rows, all non-null. If `fabric_workspace_id` is NULL, re-copy the Lakehouse URL.

---

## Step 4: Create the Catalog Integration

```sql
CREATE OR REPLACE CATALOG INTEGRATION SnowSummit_RetailRaw_IRC_INT
    CATALOG_SOURCE = ICEBERG_REST
    TABLE_FORMAT = ICEBERG
    REST_CONFIG = (
        CATALOG_URI = 'https://onelake.table.fabric.microsoft.com/iceberg'
        CATALOG_NAME = $catalog_name
    )
    REST_AUTHENTICATION = (
        TYPE = OAUTH
        OAUTH_TOKEN_URI = $oauth_token_uri
        OAUTH_CLIENT_ID = $azure_oauth_app_client_id
        OAUTH_CLIENT_SECRET = $azure_oauth_client_secret_value
        OAUTH_ALLOWED_SCOPES = ('https://storage.azure.com/.default')
    )
    ENABLED = TRUE;

SHOW CATALOG INTEGRATIONS LIKE '%IRC_INT%';
```

---

## Step 5: Create the External Volume

```sql
CREATE OR REPLACE EXTERNAL VOLUME SnowSummit_RetailRaw_EXTERNAL_VOLUME
    STORAGE_LOCATIONS = ((
        NAME = 'SnowSummit_RetailRaw_EXTERNAL_VOLUME',
        STORAGE_PROVIDER = 'AZURE',
        STORAGE_BASE_URL = $storage_base_url,
        AZURE_TENANT_ID = $azure_tenant_id
    ))
    ALLOW_WRITES = FALSE;
```

---

## Step 6: Retrieve Consent URL and Multi-Tenant App Name

```sql
DESC EXTERNAL VOLUME SnowSummit_RetailRaw_EXTERNAL_VOLUME;

SELECT
    PARSE_JSON("property_value"):AZURE_MULTI_TENANT_APP_NAME::STRING AS AZURE_MULTI_TENANT_APP_NAME,
    PARSE_JSON("property_value"):AZURE_CONSENT_URL::STRING            AS AZURE_CONSENT_URL_CLICK_ME
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "property" = 'STORAGE_LOCATION_1';
```

**Action required**:
1. **Click the `AZURE_CONSENT_URL`** -- you'll be redirected to snowflake.com (this IS the success confirmation)
2. **Copy the `AZURE_MULTI_TENANT_APP_NAME`** -- you'll need it for the next two steps

> **Note**: In Fabric's Manage Access UI, search for the service principal using just the base name (without the `_<timestamp>` suffix).

---

## Step 7: Grant Fabric Workspace Access

In **Fabric** (not the Lakehouse -- the workspace level):

1. Top-right -> **Manage Access** -> **+ Add people or groups**
2. Add your **Managed App Name** (from Step 2) -> set role to **Contributor**
3. Add the **AZURE_MULTI_TENANT_APP_NAME** (from Step 6) -> set role to **Contributor**

---

## Step 8: Create the Catalog Linked Database

```sql
CREATE OR REPLACE DATABASE SnowSummit_RetailRaw_CLDB
    LINKED_CATALOG = (
        CATALOG = SnowSummit_RetailRaw_IRC_INT
    )
    EXTERNAL_VOLUME = SnowSummit_RetailRaw_EXTERNAL_VOLUME;

-- Wait 1-2 minutes for the initial sync, then:
USE DATABASE SnowSummit_RetailRaw_CLDB;
SHOW TABLES;

-- Validate row counts
SELECT COUNT(*) AS transaction_count FROM SnowSummit_RetailRaw_CLDB."dbo"."raw_transactions";
SELECT COUNT(*) AS product_count     FROM SnowSummit_RetailRaw_CLDB."dbo"."product_catalog";
SELECT COUNT(*) AS store_count       FROM SnowSummit_RetailRaw_CLDB."dbo"."store_locations";
SELECT COUNT(*) AS customer_count    FROM SnowSummit_RetailRaw_CLDB."dbo"."customer_profiles";
```

**Expected**: ~5,000 transactions, ~253 products, ~55 stores, 500 customers.

---

## Step 9: Set Up the OneLake Write Path

### 9a. Create the Database and Warehouse

```sql
CREATE OR REPLACE WAREHOUSE SNOWSUMMIT_WH
    WAREHOUSE_SIZE = 'XSMALL'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE;

CREATE OR REPLACE DATABASE SNOWSUMMIT_ANALYTICS_ONELAKE;
```

### 9b. Create a Snowflake Connection in Fabric

1. In Fabric: **Settings gear** -> **Manage connections and gateways** -> **+ New** -> **New connection**
2. Select **Cloud** tab, fill in:
   - Connection name: `SnowSummitLab_Snowflake`
   - Connection type: **Snowflake**
   - Server: `<account>.snowflakecomputing.com`
   - Warehouse: `SNOWSUMMIT_WH`
   - Authentication: **Snowflake** (username + password)
3. Save, then open connection settings -> **Managed users** -> add the `AZURE_MULTI_TENANT_APP_NAME` with **User** access

### 9c. Connect the Database to OneLake (Snowsight Wizard)

1. In Snowsight: **Ingestion** -> **Add Data** -> **Microsoft OneLake**
2. Enter your Fabric Tenant ID -> Continue
3. Enter your Fabric Connection ID (from the connection URL in Fabric)
4. Select your workspace -> select `SNOWSUMMIT_ANALYTICS_ONELAKE` -> **Create Volume**

**Confirmation**: A Snowflake Database item should appear in your Fabric workspace.

---

## Step 10: Create Star Schema as Iceberg Tables

```sql
USE ROLE ACCOUNTADMIN;
USE DATABASE SNOWSUMMIT_ANALYTICS_ONELAKE;
USE WAREHOUSE SNOWSUMMIT_WH;

CREATE OR REPLACE ICEBERG TABLE DIM_PRODUCT CATALOG = 'SNOWFLAKE' AS
SELECT
    p."product_id"::INT AS product_key,
    p."product_name"::STRING AS product_name,
    p."category"::STRING AS category,
    p."subcategory"::STRING AS subcategory,
    p."brand"::STRING AS brand,
    p."unit_cost"::NUMBER(10,2) AS unit_cost,
    p."supplier_name"::STRING AS supplier_name,
    CASE
        WHEN p."unit_cost"::NUMBER(10,2) < 25 THEN 'Budget'
        WHEN p."unit_cost"::NUMBER(10,2) < 100 THEN 'Mid-Range'
        WHEN p."unit_cost"::NUMBER(10,2) < 300 THEN 'Premium'
        ELSE 'Luxury'
    END AS cost_tier
FROM SnowSummit_RetailRaw_CLDB."dbo"."product_catalog" p;

CREATE OR REPLACE ICEBERG TABLE DIM_STORE CATALOG = 'SNOWFLAKE' AS
SELECT
    s."store_id"::INT AS store_key,
    s."store_name"::STRING AS store_name,
    s."city"::STRING AS city,
    s."state"::STRING AS state,
    s."region"::STRING AS region,
    s."store_type"::STRING AS store_type,
    s."opening_date"::DATE AS opening_date,
    s."square_footage"::INT AS square_footage,
    CASE
        WHEN s."square_footage"::INT < 10000 THEN 'Small'
        WHEN s."square_footage"::INT < 25000 THEN 'Medium'
        ELSE 'Large'
    END AS store_size_category,
    DATEDIFF('year', s."opening_date"::DATE, CURRENT_DATE()) AS years_open
FROM SnowSummit_RetailRaw_CLDB."dbo"."store_locations" s;

CREATE OR REPLACE ICEBERG TABLE DIM_CUSTOMER CATALOG = 'SNOWFLAKE' AS
SELECT
    c."customer_id"::INT AS customer_key,
    c."first_name"::STRING AS first_name,
    c."last_name"::STRING AS last_name,
    c."first_name"::STRING || ' ' || c."last_name"::STRING AS full_name,
    c."email"::STRING AS email,
    c."signup_date"::DATE AS signup_date,
    c."loyalty_tier"::STRING AS loyalty_tier,
    c."preferred_store_id"::INT AS preferred_store_key,
    DATEDIFF('day', c."signup_date"::DATE, CURRENT_DATE()) AS days_since_signup,
    CASE
        WHEN c."loyalty_tier"::STRING = 'Platinum' THEN 4
        WHEN c."loyalty_tier"::STRING = 'Gold' THEN 3
        WHEN c."loyalty_tier"::STRING = 'Silver' THEN 2
        ELSE 1
    END AS loyalty_rank
FROM SnowSummit_RetailRaw_CLDB."dbo"."customer_profiles" c;

CREATE OR REPLACE ICEBERG TABLE DIM_DATE CATALOG = 'SNOWFLAKE' AS
WITH date_spine AS (
    SELECT DATEADD('day', seq4(), '2024-01-01'::DATE) AS date_value
    FROM TABLE(GENERATOR(ROWCOUNT => 366))
)
SELECT
    date_value AS date_key, YEAR(date_value) AS year, QUARTER(date_value) AS quarter,
    MONTH(date_value) AS month, MONTHNAME(date_value) AS month_name,
    WEEKOFYEAR(date_value) AS week_of_year, DAYOFWEEK(date_value) AS day_of_week,
    DAYNAME(date_value) AS day_name, DAY(date_value) AS day_of_month,
    CASE WHEN DAYOFWEEK(date_value) IN (0, 6) THEN TRUE ELSE FALSE END AS is_weekend,
    CONCAT('Q', QUARTER(date_value), ' ', YEAR(date_value)) AS quarter_label,
    CONCAT(MONTHNAME(date_value), ' ', YEAR(date_value)) AS month_label
FROM date_spine WHERE date_value <= '2024-12-31';

CREATE OR REPLACE ICEBERG TABLE FACT_DAILY_SALES CATALOG = 'SNOWFLAKE' AS
SELECT
    t."transaction_id"::STRING AS transaction_id,
    t."transaction_date"::DATE AS transaction_date,
    t."store_id"::INT AS store_key,
    COALESCE(TRY_CAST(t."customer_id" AS INT), -1) AS customer_key,
    t."product_id"::INT AS product_key,
    t."transaction_date"::DATE AS date_key,
    t."quantity"::INT AS quantity,
    t."unit_price"::NUMBER(10,2) AS unit_price,
    t."discount_pct"::NUMBER(5,2) AS discount_pct,
    t."payment_method"::STRING AS payment_method,
    t."cashier_id"::STRING AS cashier_id,
    (t."quantity"::INT * t."unit_price"::NUMBER(10,2)) AS gross_amount,
    ROUND((t."quantity"::INT * t."unit_price"::NUMBER(10,2)) * (t."discount_pct"::NUMBER(5,2) / 100), 2) AS discount_amount,
    ROUND((t."quantity"::INT * t."unit_price"::NUMBER(10,2)) * (1 - t."discount_pct"::NUMBER(5,2) / 100), 2) AS net_amount,
    ROUND(t."unit_price"::NUMBER(10,2) - COALESCE(p.unit_cost, 0), 2) AS unit_margin,
    ROUND(t."quantity"::INT * (t."unit_price"::NUMBER(10,2) - COALESCE(p.unit_cost, 0)), 2) AS total_margin,
    CASE WHEN t."customer_id" IS NULL OR t."customer_id"::STRING = '' THEN TRUE ELSE FALSE END AS is_walkin_customer,
    CASE WHEN t."discount_pct"::NUMBER(5,2) > 0 THEN TRUE ELSE FALSE END AS has_discount
FROM SnowSummit_RetailRaw_CLDB."dbo"."raw_transactions" t
LEFT JOIN DIM_PRODUCT p ON t."product_id"::INT = p.product_key;
```

### Validate

```sql
SELECT 'FACT_DAILY_SALES' AS table_name, COUNT(*) AS rows FROM FACT_DAILY_SALES UNION ALL
SELECT 'DIM_PRODUCT', COUNT(*) FROM DIM_PRODUCT UNION ALL
SELECT 'DIM_STORE', COUNT(*) FROM DIM_STORE UNION ALL
SELECT 'DIM_CUSTOMER', COUNT(*) FROM DIM_CUSTOMER UNION ALL
SELECT 'DIM_DATE', COUNT(*) FROM DIM_DATE
ORDER BY table_name;
```

**Check Fabric**: Your 5 Iceberg tables should now be visible in the Fabric workspace under the Snowflake Database item.

---

## Step 11: Query Across the Full Architecture

```sql
SELECT
    d.month_name, s.region, p.category,
    COUNT(f.transaction_id) AS total_transactions,
    SUM(f.net_amount) AS total_revenue,
    ROUND(SUM(f.total_margin) / NULLIF(SUM(f.net_amount), 0) * 100, 1) AS margin_pct
FROM FACT_DAILY_SALES f
JOIN DIM_DATE d ON f.date_key = d.date_key
JOIN DIM_STORE s ON f.store_key = s.store_key
JOIN DIM_PRODUCT p ON f.product_key = p.product_key
GROUP BY d.month, d.month_name, s.region, p.category
ORDER BY d.month, s.region, total_revenue DESC;
```

---

## Cleanup (Optional)

```sql
USE ROLE ACCOUNTADMIN;
DROP DATABASE IF EXISTS SNOWSUMMIT_ANALYTICS_ONELAKE;
DROP DATABASE IF EXISTS SnowSummit_RetailRaw_CLDB;
DROP CATALOG INTEGRATION IF EXISTS SnowSummit_RetailRaw_IRC_INT;
DROP EXTERNAL VOLUME IF EXISTS SnowSummit_RetailRaw_EXTERNAL_VOLUME;
DROP WAREHOUSE IF EXISTS SNOWSUMMIT_WH;
```

In Fabric: delete the `SnowSummitLab_Snowflake` connection and remove both service principal grants from workspace Manage Access.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `SHOW TABLES` empty after 2 min | Missing Manage Access grant | Re-add both service principals as Contributor |
| Service principal not found in Fabric search | Searching with `_<timestamp>` suffix | Use just the base name |
| `AZURE_CONSENT_URL` shows "Admin consent required" | Tenant admin consent policy | Ask tenant admin to approve |
| CLD query returns NULL for all columns | Missing double-quote identifiers | Use `"dbo"."table_name"` format |
| `fabric_workspace_id` NULL | Malformed Lakehouse URL | Must be from Lakehouse page: `.../groups/<guid>/lakehouses/<guid>` |
| Iceberg tables not visible in Fabric | OneLake wizard not completed | Complete Step 9c first |
| `CREATE ICEBERG TABLE` fails | No external volume | Complete Step 9c wizard first |

---

## Resources

- [CREATE CATALOG INTEGRATION (Apache Iceberg REST)](https://docs.snowflake.com/en/sql-reference/sql/create-catalog-integration-rest)
- [CREATE EXTERNAL VOLUME](https://docs.snowflake.com/en/sql-reference/sql/create-external-volume)
- [CREATE DATABASE (catalog-linked)](https://docs.snowflake.com/en/sql-reference/sql/create-database-catalog-linked)
- [Getting started with OneLake table APIs for Iceberg](https://learn.microsoft.com/en-us/fabric/onelake/table-apis/iceberg-table-apis-get-started)
- [Use Snowflake with Iceberg tables in OneLake](https://learn.microsoft.com/en-us/fabric/onelake/onelake-iceberg-snowflake)
- [Cortex Code Documentation](https://docs.snowflake.com/en/user-guide/ui-snowsight/cortex-code)

If you have any questions, reach out to your Snowflake account team!
