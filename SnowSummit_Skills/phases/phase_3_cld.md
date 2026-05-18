# Phase 3: Catalog Linked Database

> **RULES**: Present all `📖` blocks as full conversational output. STOP at every checkpoint. NEVER invent steps not in this file. NEVER summarize or skip educational content.

---

## Why This Matters

This is the first proof of interop. When this phase completes, Snowflake will be querying tables that live in Microsoft Fabric's OneLake — zero-copy, no data movement, full SQL semantics. The CLD is the object that makes Snowflake a first-class reader of OneLake data. If this works, the foundation for the entire workshop is set.

---

### 📖 What is a Catalog Linked Database (CLD)?

A **Catalog Linked Database** is a standard Snowflake database object — it has a name, you USE it, you query tables in it with normal SQL. The difference is that instead of storing data in Snowflake's own storage, it points to an external Iceberg catalog.

When you create it, Snowflake calls the Catalog Integration's IRC endpoint to discover what tables exist in your Fabric Lakehouse. Those tables are then registered in Snowflake's metadata layer as first-class objects. You can query them with `SELECT`, join them with other Snowflake tables, aggregate them — full SQL semantics.

**Zero-copy**: Snowflake's query engine reads the Parquet files directly from OneLake's ADLS storage (via the External Volume). No data is copied into Snowflake. If someone updates a table in Fabric, the next Snowflake query sees the updated data.

**Key syntax explained**:
```sql
CREATE OR REPLACE DATABASE IDENTIFIER($snowflake_cld_name)
  LINKED_CATALOG = (
    CATALOG = $snowflake_catalog_integration_name
    -- LINKED_CATALOG tells Snowflake to discover tables from the IRC, not manage them itself
    -- The CATALOG parameter points to the Catalog Integration we just created
  )
  EXTERNAL_VOLUME = $snowflake_external_volume_name;
  -- This is how Snowflake knows which storage credentials to use when reading Parquet files
```

**What happens after you run this**: Snowflake sends a request to the IRC endpoint listing all tables in the Lakehouse namespace. This sync takes 1–2 minutes. The tables will initially not appear in `SHOW TABLES` — that's normal. Wait and retry.

**Double-quoted identifiers**: Fabric uses lowercase table and schema names. Snowflake defaults to uppercase. You'll need double quotes when referencing CLD table names in queries: `"dbo"."raw_transactions"`. CoCo handles this automatically in the embedded SQL.

> ⚠️ **Common syntax error**: CLD tables use lowercase names from Fabric. Always use double quotes: `"dbo"."raw_transactions"`, not `dbo.RAW_TRANSACTIONS`.

> 🛑 **STOP** — Present the explanation above. Wait for **"go"** before executing.

---

**[EXECUTE AFTER "go"]** CoCo executes:

```sql
USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE DATABASE SnowSummit_RetailRaw_CLDB
    LINKED_CATALOG = (
        CATALOG = SnowSummit_RetailRaw_IRC_INT
    )
    EXTERNAL_VOLUME = SnowSummit_RetailRaw_EXTERNAL_VOLUME;
```

**After creation**, CoCo runs SHOW TABLES. Wait up to 2 minutes for the initial IRC sync. Expected: 4 tables under `dbo` schema — `customer_profiles`, `product_catalog`, `raw_transactions`, `store_locations`.

```sql
USE DATABASE SnowSummit_RetailRaw_CLDB;
SHOW SCHEMAS;
SHOW TABLES;
```

If tables don't appear after 2 minutes, CoCo runs the catalog link status diagnostic. Look at `error_message` — the most common cause is missing Manage Access grants from the previous STOP.

```sql
WITH catalog_link_status AS (
    SELECT PARSE_JSON(SYSTEM$CATALOG_LINK_STATUS('SnowSummit_RetailRaw_CLDB')) AS raw
)
SELECT
    cls.raw:executionState::STRING AS execution_state,
    fd.VALUE:errorCode::STRING AS error_code,
    fd.VALUE:errorMessage::STRING AS error_message,
    fd.VALUE:qualifiedEntityName::STRING AS qualified_entity_name,
    cls.raw:lastLinkAttemptStartTime::STRING AS last_link_attempt
FROM catalog_link_status cls,
    LATERAL FLATTEN(INPUT => cls.raw, PATH => 'failureDetails', OUTER => TRUE) fd;
```

> 💡 **Checkpoint** — Does everyone see 4 tables in SHOW TABLES? Say **"go"** to validate row counts.

---

**[EXECUTE AFTER "go"]** CoCo executes:

```sql
USE ROLE ACCOUNTADMIN;
USE DATABASE SnowSummit_RetailRaw_CLDB;

SELECT COUNT(*) AS transaction_count FROM SnowSummit_RetailRaw_CLDB."dbo"."raw_transactions";
SELECT COUNT(*) AS product_count     FROM SnowSummit_RetailRaw_CLDB."dbo"."product_catalog";
SELECT COUNT(*) AS store_count       FROM SnowSummit_RetailRaw_CLDB."dbo"."store_locations";
SELECT COUNT(*) AS customer_count    FROM SnowSummit_RetailRaw_CLDB."dbo"."customer_profiles";

SELECT * FROM SnowSummit_RetailRaw_CLDB."dbo"."raw_transactions" LIMIT 5;

SELECT
    "category"::STRING AS category,
    COUNT(*) AS product_count,
    ROUND(AVG("unit_cost"::FLOAT), 2) AS avg_cost
FROM SnowSummit_RetailRaw_CLDB."dbo"."product_catalog"
GROUP BY category
ORDER BY product_count DESC;
```

**Expected counts**: ~5,000 transactions, ~253 products, 500 customers, ~55 stores.

---

## 💬 DISCUSSION: What Just Happened?

Pause here and let it sink in. Key talking points:

- **"How is this different from an external table?"** External tables in Snowflake point to files you manage. A CLD points to a full Iceberg catalog managed by another system. The catalog tells Snowflake about schema evolution, new partitions, and snapshot history — not just file locations.

- **"What if someone updates data in Fabric right now?"** The next Snowflake query will see it. The CLD reflects the current Iceberg snapshot at query time.

- **"Can I write to CLD tables?"** Not in the standard pattern. CLD is a read path. To write data back to OneLake, you use Snowflake-managed Iceberg (Phase 5).

**When the room is ready, continue to Phase 4.**

---

**Phase 3 complete.** Say **"go"** to continue to Phase 4 (Star Schema Modeling).

**When the user says "go", READ `phases/phase_4_onelake_setup.md` and follow its instructions exactly.**
