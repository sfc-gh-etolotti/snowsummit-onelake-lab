# Phase 1: Collect Participant Values

> **RULES**: Present all `📖` blocks as full conversational output. STOP at every checkpoint. NEVER invent steps not in this file. NEVER summarize or skip educational content.

---

## Why This Matters

Every subsequent phase depends on these 5 values. They connect Snowflake to your specific Azure tenant, App Registration, and Fabric Lakehouse. If any value is wrong, everything downstream fails silently or with cryptic errors. Take the time to get them right now.

---

### 📖 Understanding the 5 values

You created an App Registration and a Fabric Lakehouse in Phase 0. Now we collect the 5 values Snowflake needs from those objects:

- **Azure Tenant ID** — which Azure tenant Snowflake authenticates against
- **OAuth App Client ID** — the "username" of your App Registration
- **OAuth Client Secret** — the "password" of your App Registration
- **Managed App Name** — the Enterprise App identity for Fabric workspace grants
- **Fabric Lakehouse URL** — the URL that contains your workspace and lakehouse GUIDs

**Why two service principals?**
You'll work with two distinct identities during setup:
1. **Your App Registration** (`SnowSummitLab_OAuth_Client`) — the identity you created in Phase 0. Snowflake uses this to call OneLake's Iceberg REST Catalog API.
2. **Snowflake's multi-tenant app** (`AZURE_MULTI_TENANT_APP_NAME`) — generated automatically by Snowflake when you create the External Volume in Phase 2. This is Snowflake's own service principal in your Azure tenant, used for storage-level access to ADLS Gen2.

Both need **Contributor** access to your Fabric workspace, but they serve different purposes and were created by different parties.

---

**I need 5 values. Please provide them one at a time.**

---

**Value 1 — Azure Tenant ID**
*Where to find it*: Azure Portal -> Search "App registrations" -> Select `SnowSummitLab_OAuth_Client` -> Overview tab -> **Directory (tenant) ID**

Format: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

**What it is**: Your organization's unique identifier in Azure AD. Every resource in your Azure tenant — your storage, your Fabric workspace, your App Registration — belongs to this tenant. Snowflake needs it to know which Azure AD to authenticate against when requesting tokens.

---

**Value 2 — OAuth App Client ID**
*Where to find it*: Same App Registration -> Overview tab -> **Application (client) ID**

Format: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

**What it is**: The unique identifier for *your specific App Registration*. This is the "username" half of the client credential pair. Snowflake uses this to identify itself when calling Azure's token endpoint.

---

**Value 3 — OAuth Client Secret Value**
*Where to find it*: App Registration -> **Certificates & secrets** -> **Client secrets** tab -> Value column

⚠️ This value was shown only once when you created it. If you didn't save it, you need to create a new secret.

**What it is**: The "password" half. Combined with the Client ID, it forms the credential Snowflake presents to Azure AD to prove it has the right to request access tokens for OneLake. Treat it like a password — never put it in version control.

---

**Value 4 — Managed App Name**
*Where to find it*: App Registration -> Overview tab -> **"Managed application in local directory"** link -> the display name shown there

⚠️ This is the **Enterprise App name**, NOT the App Registration name. They are different objects.

**What it is**: When you create an App Registration, Azure automatically creates a linked Enterprise Application (also called a Service Principal) in your tenant. This is the actual identity that gets granted permissions. The name you need is the display name of this Enterprise App. You'll add this as a Contributor in your Fabric workspace in Phase 2.

---

**Value 5 — Fabric Lakehouse URL**
*Where to find it*: Open your **RetailRaw** Lakehouse in Fabric -> copy the full browser URL

Format: `https://app.fabric.microsoft.com/groups/<workspace-guid>/lakehouses/<lakehouse-guid>`

**What it is**: Snowflake will parse two GUIDs out of this URL — the workspace ID and the lakehouse ID — to construct the Iceberg catalog namespace (`workspace_id/lakehouse_id`) and the Azure storage path. The exact URL format matters: it must come from the Lakehouse page, not the workspace homepage.

---

Store the 5 values internally as:
- `{{AZURE_TENANT_ID}}`
- `{{AZURE_OAUTH_APP_CLIENT_ID}}`
- `{{AZURE_OAUTH_CLIENT_SECRET_VALUE}}`
- `{{AZURE_MANAGED_APP_NAME}}`
- `{{FABRIC_LAKEHOUSE_URL}}`

---

**Phase 1 complete.** Say **"go"** to continue to Phase 2 (Snowflake Infrastructure).

**When the user says "go", READ `phases/phase_2_infrastructure.md` and follow its instructions exactly.**
