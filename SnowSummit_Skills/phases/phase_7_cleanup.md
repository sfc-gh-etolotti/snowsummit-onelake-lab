# Phase 7: Cleanup (Optional)

> **RULES**: Present all content as full conversational output. STOP at checkpoints. NEVER invent steps not in this file.

---

Ask: "Would you like to tear down all SnowSummit lab objects? This drops the warehouse, database, integrations, and external volumes."

If yes, CoCo executes:

```sql
USE ROLE ACCOUNTADMIN;

DROP DATABASE IF EXISTS SNOWSUMMIT_ANALYTICS_ONELAKE;
DROP DATABASE IF EXISTS SnowSummit_RetailRaw_CLDB;
DROP CATALOG INTEGRATION IF EXISTS SnowSummit_RetailRaw_IRC_INT;
DROP EXTERNAL VOLUME IF EXISTS SnowSummit_RetailRaw_EXTERNAL_VOLUME;
DROP WAREHOUSE IF EXISTS SNOWSUMMIT_WH;
```

If no, summarize what was built:

```
Objects created in your Snowflake account:
  Warehouse:    SNOWSUMMIT_WH
  Database:     SNOWSUMMIT_ANALYTICS_ONELAKE (Iceberg star schema — OneLake storage)
                SnowSummit_RetailRaw_CLDB (Catalog Linked Database — OneLake catalog)
  Integrations: SnowSummit_RetailRaw_IRC_INT (Catalog Integration)
                SnowSummit_RetailRaw_EXTERNAL_VOLUME (External Volume — read)
                [write-enabled volume created by the OneLake wizard]
```

Also in Fabric: delete the `SnowSummitLab_Snowflake` connection from Manage connections and gateways, and remove the two service principal grants from workspace Manage Access.

---

## Troubleshooting Reference

| Symptom | Likely cause | Fix |
|---|---|---|
| `SHOW TABLES` empty after 2 min | Missing Manage Access grant | Re-add both service principals (App Reg + multi-tenant app) as Contributor to Fabric workspace |
| Service principal not found in Fabric search | Searching with `_<timestamp>` suffix | Search for just the base name (e.g., `7uzzjmsnowflakepacint` not `7uzzjmsnowflakepacint_1713998620631`) — Fabric strips the timestamp from the display name |
| `AZURE_CONSENT_URL` shows "Admin consent required" | Tenant admin consent policy | Ask tenant admin to approve in Azure Portal -> Enterprise Applications, or use facilitator environment |
| CLD query returns NULL for all columns | Double-quote identifier missing | Use `"dbo"."table_name"` format with double quotes |
| `fabric_workspace_id` NULL after SET variables | Malformed Lakehouse URL | URL must come from the Lakehouse page, format: `.../groups/<guid>/lakehouses/<guid>` |
| Iceberg tables not visible in Fabric | OneLake connection not complete | Confirm Snowflake Database item appeared in Fabric workspace after Phase 4 STOP B |
| `CREATE ICEBERG TABLE` fails with "no external volume" | Snowsight wizard not completed | Complete Phase 4 STOP B (Snowsight Add Data -> Microsoft OneLake) before creating tables |
| Session variables NULL after Snowsight restart | Session context reset | Re-run Phase 2a SET block — variables don't persist across Snowsight sessions |

---

**Lab complete.**
