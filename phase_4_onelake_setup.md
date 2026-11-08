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

### Step 4b: Create a Fabric Service User with Key-Pair Authentication

### 📖 Why a dedicated service user instead of your own login?

Fabric's Snowflake connection signs in to Snowflake *non-interactively* — nobody is there to answer a prompt. That rules out the obvious options:

- **Your own username + password** fails if your user has MFA enforced (`AUTHENTICATOR_TOTP_REQUIRES_PASSCODE_PROGRAMMATICALLY` in login history). Fabric cannot answer an MFA challenge.
- **A Programmatic Access Token (PAT) on your own user** fails unless a network policy is attached to that user (`INCOMING_REQUEST_BLOCKED_NETWORK_POLICY_REQUIRED`). Fabric connects from changing Azure IP ranges, so pinning a network policy to them is fragile.

The robust pattern is a **dedicated service user** with **key-pair authentication**: Snowflake stores only the *public* key; Fabric holds the *private* key and signs a JWT with it at login. No password, no MFA prompt, no network policy dependency.

The service user gets a narrowly scoped role — not ACCOUNTADMIN — with just enough privilege to see the database, run on the warehouse, and read the Iceberg tables.

> 🛑 **STOP** — Present the explanation above. Wait for **"go"** before executing.

---

**[EXECUTE AFTER "go"]** CoCo executes the steps below in order.

**1. Generate a 2048-bit RSA key pair.** CoCo runs this in its sandbox and saves the files to the workspace folder `snowsummit_keys/`. If `openssl` is unavailable, use the Python fallback.

```bash
# openssl
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out rsa_key.p8 -nocrypt
openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub
```

```python
# Python fallback (cryptography library)
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization as s
k = rsa.generate_private_key(public_exponent=65537, key_size=2048)
open('rsa_key.p8','wb').write(k.private_bytes(s.Encoding.PEM, s.PrivateFormat.PKCS8, s.NoEncryption()))
open('rsa_key.pub','wb').write(k.public_key().public_bytes(s.Encoding.PEM, s.PublicFormat.SubjectPublicKeyInfo))
```

**2. Create the role, grants, and service user.** `{{RSA_PUBLIC_KEY}}` is the contents of `rsa_key.pub` with the `-----BEGIN/END PUBLIC KEY-----` lines and newlines removed.

```sql
USE ROLE ACCOUNTADMIN;

CREATE ROLE IF NOT EXISTS SNOWSUMMIT_FABRIC_ROLE;
GRANT USAGE  ON DATABASE SNOWSUMMIT_ANALYTICS_ONELAKE                        TO ROLE SNOWSUMMIT_FABRIC_ROLE;
GRANT USAGE  ON ALL SCHEMAS IN DATABASE SNOWSUMMIT_ANALYTICS_ONELAKE         TO ROLE SNOWSUMMIT_FABRIC_ROLE;
GRANT USAGE  ON FUTURE SCHEMAS IN DATABASE SNOWSUMMIT_ANALYTICS_ONELAKE      TO ROLE SNOWSUMMIT_FABRIC_ROLE;
GRANT SELECT ON ALL ICEBERG TABLES IN DATABASE SNOWSUMMIT_ANALYTICS_ONELAKE    TO ROLE SNOWSUMMIT_FABRIC_ROLE;
GRANT SELECT ON FUTURE ICEBERG TABLES IN DATABASE SNOWSUMMIT_ANALYTICS_ONELAKE TO ROLE SNOWSUMMIT_FABRIC_ROLE;
GRANT USAGE  ON WAREHOUSE SNOWSUMMIT_WH                                      TO ROLE SNOWSUMMIT_FABRIC_ROLE;

-- Lets ACCOUNTADMIN keep control of anything this role owns (needed in Step 4c)
GRANT ROLE SNOWSUMMIT_FABRIC_ROLE TO ROLE ACCOUNTADMIN;

CREATE USER IF NOT EXISTS SNOWSUMMIT_FABRIC_SVC
    TYPE = SERVICE
    LOGIN_NAME = 'SNOWSUMMIT_FABRIC_SVC'
    DISPLAY_NAME = 'Service - SnowSummit Fabric'
    DEFAULT_ROLE = SNOWSUMMIT_FABRIC_ROLE
    DEFAULT_WAREHOUSE = SNOWSUMMIT_WH
    RSA_PUBLIC_KEY = '{{RSA_PUBLIC_KEY}}';

GRANT ROLE SNOWSUMMIT_FABRIC_ROLE TO USER SNOWSUMMIT_FABRIC_SVC;

DESC USER SNOWSUMMIT_FABRIC_SVC ->> SELECT "property", "value" FROM $1
    WHERE "property" IN ('TYPE', 'RSA_PUBLIC_KEY_FP', 'DEFAULT_ROLE');
```

**Expected**: `TYPE = SERVICE`, a `RSA_PUBLIC_KEY_FP` starting with `SHA256:`, and `DEFAULT_ROLE = SNOWSUMMIT_FABRIC_ROLE`.

> ⚠️ `snowsummit_keys/rsa_key.p8` is an **unencrypted private key**. Participants should download it, upload it to Fabric in Manual Step A, then delete the folder from the workspace.

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
   - Authentication: **Key pair**
   - Username: `SNOWSUMMIT_FABRIC_SVC`
   - Private key: upload or paste `snowsummit_keys/rsa_key.p8` (include the `-----BEGIN PRIVATE KEY-----` / `-----END PRIVATE KEY-----` lines)
   - Passphrase: leave blank (the key is unencrypted)
5. Save the connection

> ⚠️ **Do not use your own user with Authentication = Snowflake.** It fails with "The credentials provided for the Snowflake source are invalid" when MFA is enforced on your user, and a PAT on your own user fails without a network policy. If the connection test fails, CoCo can check `LOGIN_HISTORY` for the exact `error_message`.

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
   > 2. Hover over the `SnowSummitLab_Snowflake` connection -> **⋯** -> **Settings**
   > 3. The **Settings** pane shows a **Connection ID** GUID near the top — copy it. (If it isn't shown, the browser URL contains it after `connections/`.)
   > 4. Paste that GUID into the Snowsight wizard
   >
   > ⚠️ The connection ID is **not** your workspace ID or Lakehouse ID. Using the wrong GUID gives: *"Cannot find valid Fabric connection UUID associated with &lt;workspace-id&gt;. Please check if the Contributor role has been granted on this resource."* The same error appears if the multi-tenant app is missing from the connection's **Managed users** or the workspace's **Manage access**.

5. Select your workspace -> Continue
6. For **Snowflake database**, select `SNOWSUMMIT_ANALYTICS_ONELAKE` -> Continue
7. Click **Create Volume** when prompted

**Confirmation**: A **Snowflake Database** item (with the Snowflake logo) should appear in your Fabric workspace. When you see it, the connection is live and Phase 5 can proceed.

> 💡 **Checkpoint** — Does everyone see the Snowflake Database item in their Fabric workspace? Say **"go"** when confirmed.

---

### Step 4c: Grant the Fabric Role Access to the Wizard's External Volume

### 📖 Why are more grants needed after the wizard?

The wizard creates a **new write-enabled External Volume**, owned by ACCOUNTADMIN, and sets it as the database's default. The Fabric service role created in Step 4b has never seen it. Fabric's sync connects as `SNOWSUMMIT_FABRIC_SVC` and runs:

```sql
SHOW PARAMETERS LIKE 'EXTERNAL_VOLUME' IN DATABASE ...;
DESC EXTERNAL VOLUME ...;
SELECT SYSTEM$VERIFY_EXTERNAL_VOLUME('...');
SHOW ICEBERG TABLES IN DATABASE ...;
```

Two separate privileges are needed:

1. **USAGE on the External Volume** — without it, `DESC EXTERNAL VOLUME` fails and Fabric shows *"Insufficient privileges to operate on external_volume ... must have USAGE granted on VOLUME ..."*
2. **OWNERSHIP of the External Volume** — `SYSTEM$VERIFY_EXTERNAL_VOLUME` requires OWNERSHIP, not just USAGE. Without it, the function does **not** fail as a query; it *returns* the string *"Error: External Volume ... does not exist or is not authorized."* Fabric surfaces that text, but query history shows every query as SUCCESS — which makes it hard to spot.

Ownership moves to `SNOWSUMMIT_FABRIC_ROLE`, and because Step 4b granted that role to ACCOUNTADMIN, admins keep full control and the Iceberg tables owned by ACCOUNTADMIN keep working. **Tradeoff**: the Fabric service user can now ALTER or DROP this volume — acceptable for the lab, worth reviewing for production.

> 🛑 **STOP** — Present the explanation above. Wait for **"go"** before executing.

---

**[EXECUTE AFTER "go"]** CoCo first finds the volume the wizard created (its name is set by the wizard, so read it rather than assume it), then substitutes it as `{{WIZARD_VOLUME}}`:

```sql
USE ROLE ACCOUNTADMIN;

SHOW PARAMETERS LIKE 'EXTERNAL_VOLUME' IN DATABASE SNOWSUMMIT_ANALYTICS_ONELAKE;
-- value column = {{WIZARD_VOLUME}}

GRANT USAGE ON EXTERNAL VOLUME {{WIZARD_VOLUME}} TO ROLE SNOWSUMMIT_FABRIC_ROLE;
GRANT OWNERSHIP ON EXTERNAL VOLUME {{WIZARD_VOLUME}}
    TO ROLE SNOWSUMMIT_FABRIC_ROLE COPY CURRENT GRANTS;

SHOW GRANTS ON EXTERNAL VOLUME {{WIZARD_VOLUME}};
```

**Expected**: OWNERSHIP and USAGE both granted to `SNOWSUMMIT_FABRIC_ROLE`.

**Verify as the Fabric role** (this is the exact call Fabric makes). Run as one anonymous block so the session returns to ACCOUNTADMIN afterwards:

```sql
EXECUTE IMMEDIATE $$
BEGIN
  USE SECONDARY ROLES NONE;
  USE ROLE SNOWSUMMIT_FABRIC_ROLE;
  LET r STRING := (SELECT SYSTEM$VERIFY_EXTERNAL_VOLUME('{{WIZARD_VOLUME}}'));
  USE ROLE ACCOUNTADMIN;
  USE SECONDARY ROLES ALL;
  RETURN r;
END;
$$;
```

**Expected**: JSON with `"success":true` and `writeResult`, `readResult`, `listResult`, `deleteResult` all `PASSED`. If it returns `Error: External Volume ... does not exist or is not authorized`, the OWNERSHIP grant did not apply.

**Then in Fabric**: open the Snowflake Database item and refresh/sync (Fabric also polls about every 5 minutes). You do not need to delete and recreate the item. Tables created in Phase 5 will appear there.

> 💡 **Checkpoint** — Did the verify call return `"success":true` for everyone? Say **"go"** when confirmed.

---

**Phase 4 complete.** Say **"go"** to continue to Phase 5 (Star Schema as Iceberg).

**When the user says "go", READ `phases/phase_5_star_schema_iceberg.md` and follow its instructions exactly.**
