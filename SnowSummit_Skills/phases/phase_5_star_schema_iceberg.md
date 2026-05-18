# Phase 5: Star Schema as Iceberg Tables

> **RULES**: Present all `📖` blocks as full conversational output. STOP at every checkpoint. NEVER invent steps not in this file. NEVER summarize or skip educational content. NEVER fabricate steps.

---

## Why This Matters

The raw Lakehouse data is flat CSV files — no relationships, no computed metrics, no analytical structure. This phase transforms that raw data into a star schema and writes it directly as Iceberg tables in OneLake. The transformation happens in Snowflake compute, and the results land as Parquet files in OneLake — instantly available in Fabric and Power BI with no Snowflake connector required. One step: read from CLD, transform, write as Iceberg.

---

### 📖 What is a star schema and what is Snowflake adding here?

A **star schema** is a data modeling pattern for analytics: a central **fact table** (transactions, events, measurements) surrounded by **dimension tables** (products, stores, customers, dates) that provide context. It makes aggregation queries fast and business reports intuitive.

The raw tables in your Fabric Lakehouse are flat CSV files — they have no dimensional model. A Power BI report built on raw transactions has to do heavy lifting on every query. A star schema pre-computes that structure.

**What Snowflake adds beyond the raw source data**:

| Table | Source | Snowflake enrichment |
|---|---|---|
| `DIM_PRODUCT` | CLD `product_catalog` | `COST_TIER` classification (Budget / Mid-Range / Premium / Luxury) based on unit cost |
| `DIM_STORE` | CLD `store_locations` | `STORE_SIZE_CATEGORY`, `YEARS_OPEN` computed from opening date |
| `DIM_CUSTOMER` | CLD `customer_profiles` | `LOYALTY_RANK` numeric score derived from tier + days since signup |
| `DIM_DATE` | **Generated in Snowflake** | Full calendar dimension using `GENERATOR(rowcount => 366)` — this table didn't exist in Fabric |
| `FACT_DAILY_SALES` | CLD `raw_transactions` + DIM_PRODUCT | `GROSS_AMOUNT`, `DISCOUNT_AMOUNT`, `NET_AMOUNT`, `UNIT_MARGIN`, `TOTAL_MARGIN` all computed |

**The pattern**: Each table is created as `CREATE ICEBERG TABLE AS SELECT` — Snowflake reads from the CLD (zero-copy reads from OneLake), applies transformations in Snowflake compute, and writes the results directly as Iceberg Parquet files to OneLake via the External Volume configured in Phase 4. The raw data stays in Fabric untouched.

### 📖 What physically happens during CREATE ICEBERG TABLE AS SELECT?

1. Snowflake reads data from the CLD tables (zero-copy from OneLake Parquet files)
2. Snowflake runs the transformation logic (joins, CASE expressions, computed columns) on Snowflake compute
3. Snowflake writes the result as **Parquet files** to the OneLake ADLS Gen2 storage path configured in the External Volume
4. Snowflake Horizon registers the new files in the **Iceberg metadata** (manifest files, snapshot JSON) — also written to OneLake storage
5. The table is immediately queryable in Snowflake AND visible as a native table in Fabric

`FACT_DAILY_SALES` takes longest (~30-60 seconds) because it has the most rows and computed columns. The DIM tables are faster. This is normal.

> 🛑 **STOP** — Present the explanation above. Wait for **"go"** before executing.

---

**[EXECUTE AFTER "go"]** CoCo executes:

```sql
USE ROLE ACCOUNTADMIN;
USE DATABASE SNOWSUMMIT_ANALYTICS_ONELAKE;
USE WAREHOUSE SNOWSUMMIT_WH;

CREATE OR REPLACE ICEBERG TABLE DIM_PRODUCT
    CATALOG = 'SNOWFLAKE'
AS
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

CREATE OR REPLACE ICEBERG TABLE DIM_STORE
    CATALOG = 'SNOWFLAKE'
AS
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

CREATE OR REPLACE ICEBERG TABLE DIM_CUSTOMER
    CATALOG = 'SNOWFLAKE'
AS
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

CREATE OR REPLACE ICEBERG TABLE DIM_DATE
    CATALOG = 'SNOWFLAKE'
AS
WITH date_spine AS (
    SELECT DATEADD('day', seq4(), '2024-01-01'::DATE) AS date_value
    FROM TABLE(GENERATOR(ROWCOUNT => 366))
)
SELECT
    date_value AS date_key,
    YEAR(date_value) AS year,
    QUARTER(date_value) AS quarter,
    MONTH(date_value) AS month,
    MONTHNAME(date_value) AS month_name,
    WEEKOFYEAR(date_value) AS week_of_year,
    DAYOFWEEK(date_value) AS day_of_week,
    DAYNAME(date_value) AS day_name,
    DAY(date_value) AS day_of_month,
    CASE WHEN DAYOFWEEK(date_value) IN (0, 6) THEN TRUE ELSE FALSE END AS is_weekend,
    CONCAT('Q', QUARTER(date_value), ' ', YEAR(date_value)) AS quarter_label,
    CONCAT(MONTHNAME(date_value), ' ', YEAR(date_value)) AS month_label
FROM date_spine
WHERE date_value <= '2024-12-31';

CREATE OR REPLACE ICEBERG TABLE FACT_DAILY_SALES
    CATALOG = 'SNOWFLAKE'
AS
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
    ROUND(
        (t."quantity"::INT * t."unit_price"::NUMBER(10,2)) * (t."discount_pct"::NUMBER(5,2) / 100), 2
    ) AS discount_amount,
    ROUND(
        (t."quantity"::INT * t."unit_price"::NUMBER(10,2)) * (1 - t."discount_pct"::NUMBER(5,2) / 100), 2
    ) AS net_amount,
    ROUND(t."unit_price"::NUMBER(10,2) - COALESCE(p.unit_cost, 0), 2) AS unit_margin,
    ROUND(t."quantity"::INT * (t."unit_price"::NUMBER(10,2) - COALESCE(p.unit_cost, 0)), 2) AS total_margin,
    CASE WHEN t."customer_id" IS NULL OR t."customer_id"::STRING = '' THEN TRUE ELSE FALSE END AS is_walkin_customer,
    CASE WHEN t."discount_pct"::NUMBER(5,2) > 0 THEN TRUE ELSE FALSE END AS has_discount
FROM SnowSummit_RetailRaw_CLDB."dbo"."raw_transactions" t
LEFT JOIN DIM_PRODUCT p ON t."product_id"::INT = p.product_key;

-- Row count verification
SELECT 'FACT_DAILY_SALES' AS iceberg_table, COUNT(*) AS row_count
FROM SNOWSUMMIT_ANALYTICS_ONELAKE.PUBLIC.FACT_DAILY_SALES UNION ALL
SELECT 'DIM_PRODUCT', COUNT(*) FROM SNOWSUMMIT_ANALYTICS_ONELAKE.PUBLIC.DIM_PRODUCT UNION ALL
SELECT 'DIM_STORE', COUNT(*) FROM SNOWSUMMIT_ANALYTICS_ONELAKE.PUBLIC.DIM_STORE UNION ALL
SELECT 'DIM_CUSTOMER', COUNT(*) FROM SNOWSUMMIT_ANALYTICS_ONELAKE.PUBLIC.DIM_CUSTOMER UNION ALL
SELECT 'DIM_DATE', COUNT(*) FROM SNOWSUMMIT_ANALYTICS_ONELAKE.PUBLIC.DIM_DATE
ORDER BY iceberg_table;

-- Business query: Monthly revenue by region and category
SELECT
    d.month_name,
    s.region,
    p.category,
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

**Expected**: 5 Iceberg tables in `SNOWSUMMIT_ANALYTICS_ONELAKE.PUBLIC`. Row counts: FACT_DAILY_SALES ~ 5,000, DIM_DATE = 366, DIM_PRODUCT ~ 253, DIM_STORE ~ 55, DIM_CUSTOMER = 500.

Ask participants: "Go to your Fabric workspace — under the Snowflake Database item, you should see DIM_CUSTOMER, DIM_DATE, DIM_PRODUCT, DIM_STORE, FACT_DAILY_SALES. Can everyone see them?"

---

**Power BI reads these natively.** Power BI uses Direct Lake mode — it reads the Parquet files from OneLake directly, the same as any other Lakehouse table. Snowflake is not in the Power BI query path.

**Optional Power BI demo**: In Fabric -> + New item -> Report -> pick `FACT_DAILY_SALES` -> build a bar chart of NET_AMOUNT by region. This report reads from OneLake, no Snowflake connection.

---

## Validate Before Continuing

- [ ] All 5 Iceberg tables visible in `SNOWSUMMIT_ANALYTICS_ONELAKE.PUBLIC`
- [ ] Tables visible in Fabric workspace under the Snowflake Database item
- [ ] Row counts match expected values
- [ ] Monthly revenue query returns 12 months of 2024 data

**Common blockers**:
- Tables not in Fabric -> Snowsight OneLake wizard not completed (Phase 4 STOP B)
- `CREATE ICEBERG TABLE` fails with "no external volume" -> Complete Phase 4 STOP B first

---

**Phase 5 complete.** Say **"go"** to continue to Phase 6 (Recap + Next Steps).

**When the user says "go", READ `phases/phase_6_recap.md` and follow its instructions exactly.**
