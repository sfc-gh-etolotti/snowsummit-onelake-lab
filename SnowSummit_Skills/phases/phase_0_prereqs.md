# Phase 0: Lab Setup

> **RULES**: Present all `📖` blocks as full conversational output. STOP at every checkpoint. NEVER invent steps not in this file. NEVER summarize or skip educational content.

---

## Why This Matters

Every step in this phase is a prerequisite for the Snowflake interop work that follows. A missing Iceberg endpoint or wrong App Registration will silently break Phase 2. We do all setup here so the rest of the lab flows without interruption.

---

### 📖 Step 1: Verify Your Snowflake Account

Before running any lab SQL, confirm you're using the `ACCOUNTADMIN` role.

**Why it's required**: Creating Catalog Integrations and External Volumes are privileged operations in Snowflake — they establish trust relationships with external systems (Azure in this case). Only `ACCOUNTADMIN` can create these objects. The warehouse and databases we create also require elevated privileges.

**In a production environment** you'd create a dedicated admin role for integration management. For this lab, `ACCOUNTADMIN` keeps things simple.

> 🛑 **STOP** — Present the explanation above. Wait for **"go"** before executing.

---

**[EXECUTE AFTER "go"]** CoCo executes:

```sql
SELECT CURRENT_ROLE(), CURRENT_ACCOUNT(), CURRENT_REGION();
```

**Expected**: `ACCOUNTADMIN` in the ROLE column. If not, run `USE ROLE ACCOUNTADMIN;` before continuing.

---

### 📖 Step 2: Create a Microsoft Fabric Workspace

You need a Fabric workspace with at least **F2 capacity**. A Fabric trial gives you F64 equivalent, which is more than enough.

**If you don't have a Fabric trial:**
1. Go to [app.fabric.microsoft.com](https://app.fabric.microsoft.com)
2. Sign in with your Microsoft account (work or school account required)
3. Click your account icon (top right) -> **Start trial** -> confirm

**Create a workspace (or use an existing one):**

> 💡 **Trial users**: Your Fabric trial comes with a **default "My workspace"** — you can use it as-is and skip the steps below. If you already have a workspace you use for testing, that works too. Just note the workspace name; you'll need it later.

To create a new workspace:
1. In the left nav, click **Workspaces** -> **+ New workspace**
2. Name it `SnowSummitLab` (or your own name — you'll reference it when setting workspace access later)
3. Expand **Advanced** -> under **License mode**, select **Trial** (or your F2+ capacity if you have one)
4. Click **Apply**

> 💡 **Checkpoint** — Does everyone have a Fabric workspace created? Say **"go"** when confirmed.

---

### 📖 Step 3: Enable External Access Tenant Settings

> ⚠️ **Critical step — requires Fabric Admin rights.** Without these two settings, Snowflake cannot connect to OneLake. Delta-to-Iceberg metadata virtualization is automatic in OneLake, but external engines like Snowflake need explicit tenant-level permission to access the data.

Two settings must be enabled in the **Fabric Admin Portal**:

1. In the upper-right corner of the Fabric UI, open **Settings** (gear icon) -> **Admin portal**
2. Under **Tenant settings**, find the **OneLake settings** section:
   - Enable **"Users can access data stored in OneLake with apps external to Fabric"**
3. In the same area, find the **Developer settings** section:
   - Enable **"Service principals can use Fabric APIs"**

> ⚠️ If you don't have Fabric Admin access, ask your tenant admin to enable these two settings before continuing. The lab will not work without them.

> 💡 **Checkpoint** — Both tenant settings enabled? Say **"go"** when confirmed.

---

### 📖 Step 4: Create the RetailRaw Lakehouse

The Lakehouse name matters — the lab SQL uses `RetailRaw` as the base name for all Snowflake objects (`SnowSummit_RetailRaw_IRC_INT`, `SnowSummit_RetailRaw_EXTERNAL_VOLUME`, `SnowSummit_RetailRaw_CLDB`).

1. In your workspace, click **+ New item**
2. Select **Lakehouse**
3. Name it exactly: **`RetailRaw`**
4. Click **Create**

You'll land on the Lakehouse Explorer. Leave this tab open.

> 💡 **Checkpoint** — RetailRaw Lakehouse created? Say **"go"** when confirmed.

---

### 📖 Step 5: Upload and Load the 4 CSV Files

The lab uses 4 datasets representing RetailEdge Inc.'s raw operational data. You need to upload them and load them as **managed Lakehouse tables** (not just files).

**The 4 files** (all are in the `DataFiles/` folder of the lab materials):

| File | Description | ~Rows |
|------|-------------|-------|
| `raw_transactions.csv` | Retail transactions, 2024 | ~5,000 |
| `product_catalog.csv` | Product master with cost and category | ~250 |
| `store_locations.csv` | 55 store locations across 5 regions | 55 |
| `customer_profiles.csv` | Loyalty program customers | 500 |

**Upload and load each file:**

1. In your `RetailRaw` Lakehouse, click **Get data** -> **Upload files**
2. Upload all 4 CSV files to the `Files` section
3. Once uploaded, for **each file**:
   - Right-click the file in the Files explorer -> **Load to Tables**
   - Select **Load** (creates a new managed Delta table)
   - The table name will match the file name (e.g., `raw_transactions`)
4. Repeat for all 4 files

> ⏱️ Each file loads in under a minute. All 4 should take about 4 minutes total.

**Verify:**
After loading, expand the **Tables** section in the Lakehouse Explorer. You should see:
- `raw_transactions`
- `product_catalog`
- `store_locations`
- `customer_profiles`

> ⚠️ If the Iceberg endpoint was enabled **after** you loaded the tables, the tables may not be exposed as Iceberg yet. Toggle the Iceberg endpoint off and on again, wait 1 minute, then refresh.

> 💡 **Checkpoint** — All 4 tables loaded? Say **"go"** when confirmed.

---

### 📖 Step 6: Copy Your Lakehouse URL

While you're in the Lakehouse, grab the URL now — you'll need it in Phase 1.

1. Make sure you're on the **Lakehouse Explorer** page for `RetailRaw` (not the workspace homepage)
2. Copy the full browser URL — it should look like:
   ```
   https://app.fabric.microsoft.com/groups/<workspace-guid>/lakehouses/<lakehouse-guid>
   ```
3. Save it somewhere — you'll paste it as **Value 5** in Phase 1

---

### 📖 Step 7: Create the Azure App Registration

### What is an Azure App Registration, and why do we need one?

**The problem**: Snowflake (a cloud service) needs to access data in Microsoft Azure (a different cloud service). These two systems don't share an identity. We need a way to give Snowflake a verifiable identity that Microsoft trusts.

**The solution**: An **Azure App Registration** is how you register an application in Microsoft's identity system (Entra ID / Azure Active Directory). When you create an App Registration, you get a **client ID + client secret** — essentially a username and password for your application. Snowflake will use these credentials to authenticate against Azure and request access tokens.

> ⏱️ Estimated time: 5 minutes. You need access to the Azure Portal for your tenant.

**Step 7a — Register the app**

1. Go to [portal.azure.com](https://portal.azure.com)
2. Search for **"App registrations"** in the top search bar -> select it
3. Click **+ New registration**
4. Fill in:
   - **Name**: `SnowSummitLab_OAuth_Client`
   - **Supported account types**: `Accounts in this organizational directory only (Single tenant)`
   - **Redirect URI**: leave blank
5. Click **Register**

You'll land on the app's Overview page. Leave this tab open — you'll need values from it.

---

**Step 7b — Add the Azure Storage API permission**

Snowflake authenticates using the scope `https://storage.azure.com/.default`. You need to register this permission on the app.

1. In the left menu, click **API permissions**
2. Click **+ Add a permission**
3. Select **Azure Storage**
4. Select **Delegated permissions**
5. Check **`user_impersonation`**
6. Click **Add permissions**
7. Click **Grant admin consent for [your tenant]** -> confirm **Yes**

> ⚠️ Admin consent requires a Global Administrator or Privileged Role Administrator. If you don't have this, ask your Azure admin — the lab will not work without it.

---

**Step 7c — Create a client secret**

1. In the left menu, click **Certificates & secrets**
2. Click **+ New client secret**
3. Add a description (e.g. `SnowSummit Lab`) and set expiry to **180 days** (or appropriate for your org)
4. Click **Add**
5. **Copy the `Value` immediately** — it is only shown once. Store it somewhere safe now.

---

**Step 7d — Note the Managed App name**

1. Go back to the app's **Overview** tab
2. Click the link next to **"Managed application in local directory"** — this opens the Enterprise Application
3. Note the **display name** shown at the top (it may differ from `SnowSummitLab_OAuth_Client`)
4. This is the identity you'll grant Contributor access to in your Fabric workspace

---

> 💡 **Checkpoint** — App Registration created, secret saved, and Lakehouse URL copied? Say **"go"** to continue to Phase 1.

---

**Phase 0 complete.** Say **"go"** to continue to Phase 1 (Collect Participant Values).

**When the user says "go", READ `phases/phase_1_collect_values.md` and follow its instructions exactly.**
