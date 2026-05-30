# Phase 4: OneLake Write-back Setup

> **RULES**: Present all `📖` blocks as full conversational output. STOP at every checkpoint. NEVER invent steps not in this file. NEVER summarize or skip educational content. NEVER fabricate steps.

---

## Why This Matters

Phase 3 proved Snowflake can READ from OneLake. Now we set up the reverse direction — Snowflake WRITING curated data back to OneLake as Iceberg. This requires a database, a Fabric connection, and a Snowsight wizard to link them together. Once this phase is complete, the write path is open and Phase 5 can create the star schema directly as Iceberg tables in OneLake.

---

### 📖 What is Snowflake-managed Iceberg, and how is it different from CLD?

In Phase 3, Fabric owned the data and Snowflake was a reader. Now, **Snowflake owns the data** — it manages the Iceberg catalog and writes Parquet files to OneLake storage.

**CLD (Phase 3)**: Snowflake connects to an *external* Iceberg catalog (OneLake's IRC). The catalog is managed by Fabric. Snowflake reads.

**Snowflake-managed Iceberg (Phase 5)**: Snowflake manages its *own* Iceberg catalog for these tables using **Snowflake Horizon** — Snowflake's unified governance and catalog layer. Horizon decides the schema, tracks snapshots, registers metadata, and maintains the full Iceberg table lifecycle. The Parquet data files are written to an External Volume (OneLake storage). Fabric sees the resulting files and reads them natively — but Snowflake Horizon controls writes, governance, and catalog management.

**The key syntax difference**:
```sql
-- CLD table (Phase 3): Snowflake is a reader of an external catalog
CREATE DATABASE ... LINKED_CATALOG = (CATALOG = SnowSummit_RetailRaw_IRC_INT)

-- Snowflake-managed Iceberg (Phase 5): Snowflake Horizon manages the catalog
CREATE ICEBERG TABLE ... CATALOG = 'SNOWFLAKE'
--                                  ^^^^^^^^^^^^
--                        'SNOWFLAKE' = Snowflake Horizon manages the Iceberg catalog
--                        vs. a catalog integration name = external catalog (e.g. OneLake IRC)
```

**Where do the files go?** When Snowflake writes to a `CATALOG = 'SNOWFLAKE'` Iceberg table, it writes Parquet files to the External Volume configured for that database — which in our case is the OneLake storage connection created by the Snowsight wizard below. The Parquet files physically land in your OneLake ADLS Gen2 storage. Fabric reads them as native Lakehouse tables.

> 🛑 **STOP** — Present the explanation above. Wait for **"go"** before continuing.

---

### Step 4a: Create the OneLake-Connected Database

### 📖 Why create the database BEFORE the wizard?

The Snowsight wizard (next step) will ask you to select which Snowflake database to connect to OneLake. That database needs to exist first so it appears in the wizard's dropdown.

> 🛑 **STOP** — Present the explanation above. Wait for **"go"** before executing.

---

**[EXECUTE AFTER "go"]** CoCo executes:

```sql
USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE WAREHOUSE SNOWSUMMIT_WH
    WAREHOUSE_SIZE = 'XSMALL'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE;

CREATE OR REPLACE DATABASE SNOWSUMMIT_ANALYTICS_ONELAKE;
```

---

## ⏸ STOP — Two Manual Setup Steps (~8 min)

### 📖 Manual Step A: Create the Snowflake Connection in Fabric

### Why does Fabric need a Snowflake connection?

When Snowflake writes Iceberg tables to OneLake, it needs to authenticate against the OneLake storage with write permissions. The Snowsight wizard (Step B below) orchestrates this, but it requires a **Snowflake connection object registered in Fabric** first — this is how Fabric's connection manager tracks which Snowflake account is authorized to write to your workspace.

Think of it as: Fabric keeps a registry of trusted connections. Before it lets Snowflake write Parquet files into its storage, it needs Snowflake to identify itself via this registered connection.

**What to do in Fabric**:
1. **Settings gear** (top right) -> **Manage connections and gateways**
2. **+ New** -> **New connection**
3. At the top of the dialog, select **Cloud** (you'll see four tabs: On-premises, Virtual network, Streaming virtual network, Cloud — Cloud must be selected or the Snowflake connection type won't be available)
4. Fill in:
   - Connection name: `SnowSummitLab_Snowflake`
   - Connection type: **Snowflake**
   - Server: `<orgname>-<account_name>.snowflakecomputing.com` (no `https://` — find this in Snowsight: click your account name bottom-left -> copy the org-qualified account identifier, e.g. `myorg-myaccount.snowflakecomputing.com`)
   - Warehouse: `SNOWSUMMIT_WH`
   - Authentication: **Snowflake** (username + password)
4. Save the connection

After saving, open the connection's settings -> **Managed users** -> add the `AZURE_MULTI_TENANT_APP_NAME` value (from Phase 2d) with **User** access.

> ⚠️ **Search tip**: When searching for the multi-tenant app in Fabric's managed users dialog, use the **base name without the `_<timestamp>` suffix**. For example, search for `7uzzjmsnowflakepacint` — not `7uzzjmsnowflakepacint_1713998620631`. Fabric's Entra ID search does not include the timestamp portion.

**Why add the multi-tenant app to managed users?** The write path to OneLake uses Snowflake's service principal identity (the multi-tenant app). Adding it to the connection's managed users tells Fabric: "this service principal is authorized to use this Snowflake connection."

> 💡 **Checkpoint** — Has everyone created the connection and added the multi-tenant app as a managed user? Say **"go"** when confirmed.

---

### 📖 Manual Step B: Connect the Database to OneLake in Snowsight

### What is this wizard doing?

The Snowsight "Add Data > Microsoft OneLake" wizard is a GUI flow that:
1. Authenticates your Snowflake account against your Fabric tenant
2. Creates a **write-enabled External Volume** in Snowflake pointing to your OneLake workspace
3. Links the `SNOWSUMMIT_ANALYTICS_ONELAKE` database to that External Volume
4. Registers a **Snowflake Database** item in your Fabric workspace that tracks this connection

This is the step that enables `CATALOG = 'SNOWFLAKE'` Iceberg tables in that database to write Parquet files to OneLake. Without it, the `CREATE ICEBERG TABLE` commands in Phase 5 will fail.

**What to do in Snowsight**:
1. Left nav -> **Ingestion** -> **Add Data**
2. Select **Microsoft OneLake**
3. Enter your **Fabric Tenant ID** (same as `{{AZURE_TENANT_ID}}`) -> Continue
4. When prompted for the **Fabric Connection ID**, enter the ID of the `SnowSummitLab_Snowflake` connection you created in Step A

   > **How to find your Fabric Connection ID:**
   > 1. In Fabric, go to **Settings gear** -> **Manage connections and gateways**
   > 2. Click on the `SnowSummitLab_Snowflake` connection to open its details
   > 3. Look at the browser URL — it contains the connection ID as a GUID:
   >    `…/connections/<connection-id>/…`
   > 4. Copy that GUID and paste it into the Snowsight wizard

5. Select your workspace -> Continue
6. For **Snowflake database**, select `SNOWSUMMIT_ANALYTICS_ONELAKE` -> Continue
7. Click **Create Volume** when prompted

**Confirmation**: A **Snowflake Database** item (with the Snowflake logo) should appear in your Fabric workspace. When you see it, the connection is live and Phase 5 can proceed.

> 💡 **Checkpoint** — Does everyone see the Snowflake Database item in their Fabric workspace? Say **"go"** when confirmed.

---

**Phase 4 complete.** Say **"go"** to continue to Phase 5 (Star Schema as Iceberg).

**When the user says "go", READ `phases/phase_5_star_schema_iceberg.md` and follow its instructions exactly.**
