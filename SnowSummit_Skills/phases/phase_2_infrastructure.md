# Phase 2: Snowflake Infrastructure

> **RULES**: Present all `📖` blocks as full conversational output. STOP at every checkpoint. NEVER invent steps not in this file. NEVER summarize or skip educational content.

---

## Why This Matters

Before Snowflake can read a single row from OneLake, it needs two infrastructure objects: a **Catalog Integration** (to discover what tables exist) and an **External Volume** (to access the storage where data files live). These are Snowflake's credentials for OneLake — without them, there's no interop. This phase also retrieves Snowflake's auto-generated service principal and grants it access to your Fabric workspace. Every step here is a prerequisite for the CLD in Phase 3.

---

### Step 2a: Set Session Variables

### 📖 What are we doing with these variables?

Snowflake doesn't have native string interpolation in SQL, but it has **session variables** — named values that persist for the duration of your connection and can be referenced with `$variable_name`.

We set the 5 raw values you just provided, then derive additional values from them automatically:

- `$fabric_workspace_id` and `$fabric_data_item_id` — parsed from the Lakehouse URL using `REGEXP_SUBSTR`. If either comes back NULL, the URL format is wrong.
- `$catalog_name` — combined as `workspace_id/lakehouse_id`, which is the Iceberg catalog namespace that identifies your specific Lakehouse on OneLake.
- `$oauth_token_uri` — assembled as `https://login.microsoftonline.com/<tenant_id>/oauth2/v2.0/token`, the Azure AD endpoint Snowflake calls to get OAuth tokens.
- `$storage_base_url` — assembled as `azure://onelake.dfs.fabric.microsoft.com/<workspace_id>/<lakehouse_id>`, the ADLS Gen2 DFS endpoint for your Lakehouse storage.

The validation query at the bottom checks all 12 variables are non-null. **If `fabric_workspace_id` or `fabric_data_item_id` is NULL, stop — the Lakehouse URL needs to be corrected before anything else will work.**

> 🛑 **STOP** — Present the explanation above. Wait for **"go"** before executing.

---

**[EXECUTE AFTER "go"]** CoCo executes the following with your values substituted:

```sql
USE ROLE ACCOUNTADMIN;

SET azure_tenant_id                  = '{{AZURE_TENANT_ID}}';
SET azure_oauth_app_client_id        = '{{AZURE_OAUTH_APP_CLIENT_ID}}';
SET azure_oauth_client_secret_value  = '{{AZURE_OAUTH_CLIENT_SECRET_VALUE}}';
SET azure_oauth_app_managed_app_name = '{{AZURE_MANAGED_APP_NAME}}';
SET fabric_lakehouse_url             = '{{FABRIC_LAKEHOUSE_URL}}';
SET fabric_lakehouse_name            = 'RetailRaw';

SET snowflake_catalog_integration_name = (SELECT 'SnowSummit_' || $fabric_lakehouse_name || '_IRC_INT');
SET snowflake_external_volume_name     = (SELECT 'SnowSummit_' || $fabric_lakehouse_name || '_EXTERNAL_VOLUME');
SET snowflake_cld_name                 = (SELECT 'SnowSummit_' || $fabric_lakehouse_name || '_CLDB');

SET fabric_workspace_id = (
    SELECT REGEXP_SUBSTR(
        $fabric_lakehouse_url,
        'groups/([0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})',
        1, 1, 'e', 1
    )
);
SET fabric_data_item_id = (
    SELECT REGEXP_SUBSTR(
        $fabric_lakehouse_url,
        'lakehouses/([0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})',
        1, 1, 'e', 1
    )
);
SET catalog_name     = (SELECT CONCAT_WS('/', $fabric_workspace_id, $fabric_data_item_id));
SET oauth_token_uri  = (SELECT CONCAT('https://login.microsoftonline.com/', $azure_tenant_id, '/oauth2/v2.0/token'));
SET storage_base_url = (SELECT CONCAT('azure://onelake.dfs.fabric.microsoft.com/', $catalog_name));

WITH validate AS (
    SELECT 'azure_tenant_id'                   AS variable_name, $azure_tenant_id                  AS value UNION ALL
    SELECT 'azure_oauth_app_client_id',                          $azure_oauth_app_client_id                 UNION ALL
    SELECT 'azure_oauth_client_secret_value',  LEFT($azure_oauth_client_secret_value, 4) || '****'          UNION ALL
    SELECT 'azure_oauth_app_managed_app_name',                   $azure_oauth_app_managed_app_name          UNION ALL
    SELECT 'fabric_workspace_id',                                $fabric_workspace_id                       UNION ALL
    SELECT 'fabric_data_item_id',                                $fabric_data_item_id                       UNION ALL
    SELECT 'catalog_name',                                       $catalog_name                              UNION ALL
    SELECT 'oauth_token_uri',                                    $oauth_token_uri                           UNION ALL
    SELECT 'storage_base_url',                                   $storage_base_url                          UNION ALL
    SELECT 'snowflake_catalog_integration_name',                 $snowflake_catalog_integration_name        UNION ALL
    SELECT 'snowflake_external_volume_name',                     $snowflake_external_volume_name            UNION ALL
    SELECT 'snowflake_cld_name',                                 $snowflake_cld_name
)
SELECT * FROM validate;
```

**Expected**: 12 rows, all values non-null. If `fabric_workspace_id` or `fabric_data_item_id` is NULL, stop — the Lakehouse URL is malformed. Ask the participant to re-copy it from the Lakehouse browser URL.

---

### Step 2b: Create the Catalog Integration

### 📖 What is a Catalog Integration?

A **Catalog Integration** is a named Snowflake object that stores the connection details for an **Iceberg REST Catalog (IRC)** endpoint.

**What is an Iceberg REST Catalog?** It's a standard REST API (defined by the Apache Iceberg open specification) that serves table metadata: what tables exist, what their schemas are, where their Parquet data files are located, and what the current snapshot is. Think of it as a *table of contents* for an Iceberg data lake.

**OneLake's role**: Microsoft OneLake natively exposes an IRC endpoint at `https://onelake.table.fabric.microsoft.com/iceberg`. Any engine that speaks the Iceberg REST spec — including Snowflake — can connect to it. This is not a proprietary integration. Snowflake is using the same open protocol it would use to connect to AWS Glue, Databricks Unity Catalog, or any other IRC-compatible catalog.

**Authentication**: Snowflake calls the IRC endpoint using an **OAuth 2.0 client credentials** grant. It presents your App Registration's Client ID and Secret to Azure AD, receives a short-lived access token, and includes that token on every call to the IRC endpoint. The scope `https://storage.azure.com/.default` grants access to Azure storage services — needed because OneLake sits on ADLS Gen2.

**Key parameters in the SQL**:
```
CATALOG_SOURCE = ICEBERG_REST          -> use the open Iceberg REST protocol
TABLE_FORMAT = ICEBERG                 -> tables are Iceberg format (not Delta)
CATALOG_URI = '...'                    -> OneLake's global IRC endpoint
CATALOG_NAME = $catalog_name           -> workspace_id/lakehouse_id — identifies YOUR Lakehouse
OAUTH_TOKEN_URI = $oauth_token_uri     -> Azure AD token endpoint for your tenant
OAUTH_CLIENT_ID = '...'               -> your App Registration's Client ID
OAUTH_CLIENT_SECRET = '...'           -> your App Registration's Secret
OAUTH_ALLOWED_SCOPES = ('https://storage.azure.com/.default')
                                       -> grants access to Azure storage services
```

> 🛑 **STOP** — Present the explanation above. Wait for **"go"** before executing.

---

**[EXECUTE AFTER "go"]** CoCo executes:

```sql
USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE CATALOG INTEGRATION SnowSummit_RetailRaw_IRC_INT
    CATALOG_SOURCE = ICEBERG_REST
    TABLE_FORMAT = ICEBERG
    REST_CONFIG = (
        CATALOG_URI = 'https://onelake.table.fabric.microsoft.com/iceberg'
        CATALOG_NAME = '{{CATALOG_NAME}}'
    )
    REST_AUTHENTICATION = (
        TYPE = OAUTH
        OAUTH_TOKEN_URI = '{{OAUTH_TOKEN_URI}}'
        OAUTH_CLIENT_ID = '{{AZURE_OAUTH_APP_CLIENT_ID}}'
        OAUTH_CLIENT_SECRET = '{{AZURE_OAUTH_CLIENT_SECRET_VALUE}}'
        OAUTH_ALLOWED_SCOPES = ('https://storage.azure.com/.default')
    )
    ENABLED = TRUE;

SHOW CATALOG INTEGRATIONS LIKE '%IRC_INT%';
```

**Expected**: One row with `enabled = true` and `catalog_source = ICEBERG_REST`.

---

### Step 2c: Create the External Volume

### 📖 What is an External Volume, and why is it separate from the Catalog Integration?

This is the most common question at this step, so let's address it directly.

**The Catalog Integration** answers: *"What tables exist and where are their metadata files?"* It calls the IRC API. It reads JSON metadata.

**The External Volume** answers: *"Can Snowflake actually touch the underlying storage?"* It gives Snowflake direct access to the **Azure Data Lake Storage Gen2** layer where the Parquet files physically live.

They connect to different systems:
- Catalog Integration -> `onelake.table.fabric.microsoft.com` (metadata REST API)
- External Volume -> `onelake.dfs.fabric.microsoft.com` (ADLS Gen2 DFS endpoint, actual file storage)

**Why both?** The Catalog Integration tells Snowflake *where* the files are. The External Volume gives Snowflake *permission to read and write* those files. You need both to query or write Iceberg data.

**`ALLOW_WRITES = FALSE`** in this script: We're creating the External Volume in read-only mode for the CLD setup. In Step 5, when we write Iceberg tables back to OneLake, we'll need write permission — but that's handled by the Snowsight wizard which creates a separate write-enabled volume.

**`AZURE_TENANT_ID`** in the storage location: This tells Snowflake which Azure tenant to authenticate against when requesting access to this storage path. It's used to generate the AZURE_CONSENT_URL and AZURE_MULTI_TENANT_APP_NAME that you'll need in the next step.

**Key parameters**:
```
STORAGE_PROVIDER = 'AZURE'              -> Azure Data Lake Storage
STORAGE_BASE_URL = $storage_base_url    -> azure://onelake.dfs.fabric.microsoft.com/<workspace>/<lakehouse>
                                         -> this is the ADLS DFS endpoint for your specific Lakehouse
AZURE_TENANT_ID = $azure_tenant_id      -> your Azure tenant — used to generate auth artifacts
ALLOW_WRITES = FALSE                    -> read-only for Phase 3; Phase 5 uses a write-enabled volume
```

> 🛑 **STOP** — Present the explanation above. Wait for **"go"** before executing.

---

**[EXECUTE AFTER "go"]** CoCo executes:

```sql
USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE EXTERNAL VOLUME SnowSummit_RetailRaw_EXTERNAL_VOLUME
    STORAGE_LOCATIONS = ((
        NAME = 'SnowSummit_RetailRaw_EXTERNAL_VOLUME',
        STORAGE_PROVIDER = 'AZURE',
        STORAGE_BASE_URL = '{{STORAGE_BASE_URL}}',
        AZURE_TENANT_ID = '{{AZURE_TENANT_ID}}'
    ))
    ALLOW_WRITES = FALSE;
```

**Expected**: `External volume RETAILRAW_EXTERNAL_VOLUME successfully created.`

---

### Step 2d: Retrieve the Consent URL and Multi-Tenant App Name

### 📖 What are these two values, and why do we need them?

When you create an External Volume, Snowflake doesn't just store configuration — it registers its own service principal in your Azure tenant and generates two critical values that you must retrieve before continuing. The way to get them is by running `DESC EXTERNAL VOLUME`, which inspects the object Snowflake just created and surfaces its generated Azure metadata.

**Why DESC EXTERNAL VOLUME?** The consent URL and multi-tenant app name don't exist until the External Volume is created — Snowflake generates them at creation time and stores them on the object. `DESC EXTERNAL VOLUME` is the only way to retrieve them.

**AZURE_CONSENT_URL**
When Snowflake creates the External Volume, it registers its own service principal in your Azure tenant — called the **multi-tenant app**. Before this service principal can access your storage, an Azure administrator must explicitly consent to its permissions. The consent URL is a pre-built Azure AD authorization URL that triggers this consent flow. Clicking it and being redirected to `snowflake.com` means the consent was granted. If you see "Admin consent required", your Azure tenant has stricter policies and a tenant admin must approve it.

**AZURE_MULTI_TENANT_APP_NAME**
This is the display name of **Snowflake's service principal** that was registered in your Azure tenant when the External Volume was created. It's a different identity from your App Registration — Snowflake generated it automatically.

> ⚠️ **Important: Fabric name vs. DESC output name.** The DESC output shows the full value including a `_<timestamp>` suffix (e.g., `7uzzjmsnowflakepacint_1713998620631`). **In Fabric's Manage Access and connection settings**, the service principal appears under just the **base name without the suffix** (e.g., `7uzzjmsnowflakepacint`). When searching in Fabric, use the base name — Fabric's Entra ID search does not include the timestamp portion. Copy both the full value (for reference) and note the base name (for Fabric search).

> 🛑 **STOP** — Present the explanation above. Wait for **"go"** before executing.

> ⚠️ **Save both values now** — you will need the multi-tenant app name in both manual STOP gates ahead.

---

**[EXECUTE AFTER "go"]** CoCo executes:

```sql
USE ROLE ACCOUNTADMIN;

DESC EXTERNAL VOLUME SnowSummit_RetailRaw_EXTERNAL_VOLUME;

SELECT
    PARSE_JSON("property_value"):AZURE_MULTI_TENANT_APP_NAME::STRING AS AZURE_MULTI_TENANT_APP_NAME,
    PARSE_JSON("property_value"):AZURE_CONSENT_URL::STRING            AS AZURE_CONSENT_URL_CLICK_ME,
    PARSE_JSON("property_value"):AZURE_TENANT_ID::STRING              AS AZURE_TENANT_ID,
    PARSE_JSON("property_value"):STORAGE_PROVIDER::STRING             AS STORAGE_PROVIDER,
    PARSE_JSON("property_value"):STORAGE_REGION::STRING               AS STORAGE_REGION
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "property" = 'STORAGE_LOCATION_1';
```

Copy both values from the result before clicking "go" on the next step.

---

## ⏸ STOP — Two Manual Steps Before CLD Creation (~5 min)

### 📖 Manual Step A: Click the Consent URL

### What is OAuth Consent and why is it needed?

OAuth consent is Microsoft's mechanism for explicitly authorizing a third-party application (Snowflake's multi-tenant app) to access resources in your tenant on behalf of a user or service. Without this consent, Snowflake's service principal exists in your tenant but has no granted permissions — it can't read or write anything.

**What to do**:
1. Open the `AZURE_CONSENT_URL` value in a browser
2. You'll be redirected to `snowflake.com` — this redirect IS the success confirmation
3. There's nothing to click or confirm at the destination — the redirect itself means the consent token was issued

**If you see "Admin consent required"**: Your Azure tenant has a policy requiring tenant admin approval for third-party app consent. Options:
- Ask a tenant admin to approve the app in Azure Portal -> Enterprise Applications -> SnowSummitLab -> Permissions -> Grant admin consent
- Use the facilitator's pre-built environment

> 💡 **Checkpoint** — Has everyone successfully clicked the consent URL and been redirected to snowflake.com? Say **"go"** when confirmed.

---

### 📖 Manual Step B: Grant Fabric Workspace Access to Both Service Principals

### Why does the Fabric workspace need these grants?

OneLake enforces access control at the **Fabric workspace level**, not just at the Azure storage level. Even though Snowflake's service principals have Azure AD credentials, Fabric requires them to also be explicitly added as workspace members before they can access the Lakehouse data.

**You need to add TWO service principals**, because two different Snowflake identities need workspace access:
1. **Your App Registration** (`{{AZURE_MANAGED_APP_NAME}}`) — the identity Snowflake uses to call the IRC metadata API
2. **Snowflake's multi-tenant app** (the `AZURE_MULTI_TENANT_APP_NAME` from the DESC output) — the identity Snowflake uses to access ADLS storage directly

Both need **Contributor** role (not Viewer — Contributor allows the storage-level read operations).

**What to do in Fabric**:
1. Open your Fabric **Workspace** (the top level — not the Lakehouse inside it)
2. Top-right corner -> **Manage Access**
3. Click **+ Add people or groups**
4. Search for and add `{{AZURE_MANAGED_APP_NAME}}` -> set role to **Contributor** -> Add
5. Repeat for the `AZURE_MULTI_TENANT_APP_NAME` value from the DESC output

> ⚠️ **Search tip**: When searching in Fabric's "Add people or groups" dialog, use the **base name without the `_<timestamp>` suffix**. For example, search for `7uzzjmsnowflakepacint` — not `7uzzjmsnowflakepacint_1713998620631`. Fabric's Entra ID directory listing does not include the timestamp portion.

**Expected result**: Your Manage Access list should show 3 entries total — your user account + both service principals as Contributor.

> 💡 **Checkpoint** — Does everyone have both service principals added as Contributor in their workspace Manage Access? This is the #1 cause of CLD failures. Confirm before continuing. Say **"go"** when all participants are ready.

---

**Phase 2 complete.** Say **"go"** to continue to Phase 3 (Catalog Linked Database).

**When the user says "go", READ `phases/phase_3_cld.md` and follow its instructions exactly.**
