# Building an Iceberg Lakehouse with Snowflake and Microsoft OneLake

## Overview

Data architectures are consolidating around open table formats and flexible compute engines. As data volumes grow and AI workloads become more demanding, maintaining multiple copies of the same datasets across systems is no longer practical. The combination of OneLake storage, Apache Iceberg tables, and Snowflake's execution engine provides a clean, sustainable model that meets the needs of this architecture.

Apache Iceberg serves as the shared table layer between Snowflake and Microsoft Fabric. Iceberg's metadata structure, versioning, and concurrency model allow multiple engines to read and write the same tables without conflict. This removes the need for synchronization pipelines, custom connectors, or duplicated storage zones. The data lives once in OneLake, and Snowflake can operate on it directly through the Iceberg REST catalog.

This lab demonstrates the full bi-directional workflow: uploading raw retail data to a Fabric Lakehouse, reading it zero-copy in Snowflake through a Catalog Linked Database, transforming it into a star schema, and writing the curated results back to OneLake as Snowflake-managed Iceberg tables -- instantly available in Power BI and other Fabric services with no Snowflake connector required.

**This lab is powered by [Cortex Code (CoCo)](https://docs.snowflake.com/en/user-guide/ui-snowsight/cortex-code)** -- Snowflake's built-in IDE. CoCo explains every concept before executing SQL, pauses for your acknowledgement at each step, and guides you through the manual Fabric/Azure configuration steps. Nothing executes without a "go".

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
- Querying the same data from both Snowflake and Fabric without data duplication

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

### What You Will Need

- A free [Snowflake Account](https://signup.snowflake.com/)
- [Fabric Capacity](https://learn.microsoft.com/en-us/fabric/get-started/fabric-trial) (trial works)
- The CSV files from the `DataFiles/` folder in this repository
- For the sake of the lab it is best if both platforms have access to the public internet and are not in a virtual network

---

## How to Run This Lab

This lab is delivered through **Cortex Code (CoCo)**, Snowflake's built-in IDE in Snowsight. CoCo reads a skill file that contains educational content and SQL -- it explains each concept, waits for your acknowledgement, then executes the SQL.

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

Type **"snowsummit"** in CoCo and say **"go"** to begin. CoCo will walk you through each phase, explaining concepts before executing any SQL. Say **"go"** to advance through each step.

---

## Lab Phases

| Phase | What Happens | Est. Time |
|-------|-------------|-----------|
| 0 | Fabric workspace setup, Lakehouse creation, CSV upload, App Registration | 20 min |
| 1 | Collect 5 Azure/Fabric values for Snowflake configuration | 5 min |
| 2 | Snowflake infrastructure: session variables, Catalog Integration, External Volume, OAuth consent, service principal grants | 15 min |
| 3 | Catalog Linked Database creation + zero-copy query validation | 10 min |
| 4 | OneLake write-back setup: database, Fabric connection, Snowsight wizard | 15 min |
| 5 | Star schema modeled directly as Iceberg tables in OneLake | 10 min |
| 6 | Recap of what was built + next steps (Power BI, Cortex AI, Copilot Studio) | 5 min |
| 7 | Cleanup + troubleshooting reference | 5 min |

**Total estimated time: ~90 minutes**

---

## Phase Details

### Phase 0: Lab Setup
Create a Microsoft Fabric workspace, enable the required tenant admin settings (external access + service principal APIs), create a `RetailRaw` Lakehouse, upload 4 CSV files from the `DataFiles/` folder, and create an Azure App Registration with a client secret. Everything needed before Snowflake touches OneLake.

### Phase 1: Collect Values
Gather the 5 values Snowflake needs: Azure Tenant ID, OAuth App Client ID, OAuth Client Secret, Managed App Name, and Fabric Lakehouse URL. CoCo explains what each value is and where to find it.

### Phase 2: Snowflake Infrastructure
Create the two Snowflake objects that connect to OneLake: a **Catalog Integration** (calls the Iceberg REST Catalog API to discover tables) and an **External Volume** (accesses the ADLS Gen2 storage where Parquet files live). Then retrieve Snowflake's auto-generated service principal, click the OAuth consent URL, and grant both service principals Contributor access to your Fabric workspace.

### Phase 3: Catalog Linked Database
Create a Catalog Linked Database (CLD) that registers OneLake Lakehouse tables as first-class Snowflake objects. Validate with row counts and sample queries. The CLD reads Parquet files directly from OneLake -- zero data movement.

### Phase 4: OneLake Write-back Setup
Create the `SNOWSUMMIT_ANALYTICS_ONELAKE` database, set up a Snowflake connection in Fabric, and run the Snowsight "Add Data > Microsoft OneLake" wizard to link the database to OneLake with write permissions. This enables Snowflake-managed Iceberg tables managed by Snowflake Horizon.

### Phase 5: Star Schema as Iceberg
Transform the raw CLD data into a star schema (DIM_PRODUCT, DIM_STORE, DIM_CUSTOMER, DIM_DATE, FACT_DAILY_SALES) using `CREATE ICEBERG TABLE AS SELECT` -- reading from OneLake, transforming in Snowflake compute, and writing Parquet files directly back to OneLake. One step, no intermediate database. The tables are immediately visible in Fabric.

### Phase 6: Recap + What's Next
Review what was built end-to-end and explore next steps: Power BI Direct Lake reports, Cortex AI enrichment, Cortex Analyst + Copilot Studio integration, and incremental pipelines with Streams + Tasks.

### Phase 7: Cleanup
Optional teardown of all Snowflake objects (warehouse, databases, integrations, external volumes) and Fabric connection cleanup. Includes a troubleshooting reference table for common issues.

---

## Repository Structure

```
snowsummit-onelake-lab/
  README.md                                     <- This quickstart guide
  SnowSummit_Skills/                            <- Upload this folder to CoCo
    SKILL.md                                    <- Master skill (routing + behavioral rules)
    phases/
      phase_0_prereqs.md                        <- Fabric setup + CSV upload + App Registration
      phase_1_collect_values.md                 <- 5 values collection
      phase_2_infrastructure.md                 <- Catalog Integration + External Volume + SP grants
      phase_3_cld.md                            <- Catalog Linked Database + validation
      phase_4_onelake_setup.md                  <- OneLake DB + Fabric connection + Snowsight wizard
      phase_5_star_schema_iceberg.md            <- Star schema CTAS directly into Iceberg
      phase_6_recap.md                          <- Recap + next steps
      phase_7_cleanup.md                        <- Cleanup + troubleshooting
  DataFiles/
    raw_transactions.csv                        <- ~5,000 retail transactions (2024)
    product_catalog.csv                         <- ~250 products with cost and category
    store_locations.csv                         <- 55 stores across 5 regions
    customer_profiles.csv                       <- 500 loyalty program customers
```

---

## Conclusion and Resources

After completing this lab, you will have built a fully bi-directional open Lakehouse:

- **Snowflake reads from OneLake** via a Catalog Linked Database (zero-copy, Iceberg REST Catalog)
- **Snowflake writes to OneLake** via Snowflake-managed Iceberg tables (Parquet files in ADLS Gen2)
- **Fabric reads from Snowflake's tables** natively via Direct Lake mode -- no connector required
- **One copy of data**, two compute engines, open Iceberg format throughout

### What You Learned

- Creating an External Volume and Catalog Integration to read Iceberg data in OneLake
- Creating a Snowflake Database in OneLake and writing Snowflake-managed Iceberg tables
- Setting OneLake as a Catalog Linked Database in Snowflake
- Transforming raw data into a star schema directly as Iceberg tables
- Querying the same data from both Snowflake and Fabric without data duplication

### Resources

- [CREATE CATALOG INTEGRATION (Apache Iceberg REST)](https://docs.snowflake.com/en/sql-reference/sql/create-catalog-integration-rest)
- [CREATE EXTERNAL VOLUME](https://docs.snowflake.com/en/sql-reference/sql/create-external-volume)
- [CREATE DATABASE (catalog-linked)](https://docs.snowflake.com/en/sql-reference/sql/create-database-catalog-linked)
- [Getting started with OneLake table APIs for Iceberg](https://learn.microsoft.com/en-us/fabric/onelake/table-apis/iceberg-table-apis-get-started)
- [Use Snowflake with Iceberg tables in OneLake](https://learn.microsoft.com/en-us/fabric/onelake/onelake-iceberg-snowflake)
- [Cortex Code Documentation](https://docs.snowflake.com/en/user-guide/ui-snowsight/cortex-code)

If you have any questions, reach out to your Snowflake account team!
