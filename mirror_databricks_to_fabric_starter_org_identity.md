# Mirror Azure Databricks into Microsoft Fabric

A step-by-step guide for connecting a Databricks Unity Catalog to Microsoft Fabric so tables appear live in Fabric's OneLake — no data copying, no ETL, no scheduled sync.

---

## What this achieves

Fabric reads the Delta files that back your Databricks tables directly from ADLS. That means:

- Queries in Fabric hit the same data your Databricks users see
- Any insert / update / delete in Databricks flows through automatically (via the Delta transaction log)
- You can build Power BI reports, Fabric notebooks, lakehouse shortcuts, and Data Factory pipelines on top of Databricks data without moving or duplicating it
- No SQL warehouse or compute needs to be running in Databricks for Fabric to read the mirrored data

## Prerequisites (get these lined up first)

| Item | Why | Who provides it |
|---|---|---|
| Azure Databricks workspace with Unity Catalog enabled | Mirroring only supports UC tables, not `hive_metastore` | You / Azure admin |
| Microsoft Fabric capacity (F SKU or trial) | Required to create the mirrored item | Fabric admin |
| At least one Delta table in a UC catalog | The thing being mirrored | You |
| Metastore admin rights in Databricks | Needed to flip one specific setting | Databricks admin |
| Ability to grant privileges on your schema | Needed for the `EXTERNAL USE SCHEMA` grant | You (if you own the schema) or Databricks admin |
| Same Microsoft Entra tenant across both | Fabric uses your Entra login to connect | Should be true by default |

## Permissions to request

### From your Databricks administrator

Send them this list:

1. **Metastore admin** (or the specific ability to enable external data access on the metastore). This is not a workspace admin role — it's a Unity Catalog role. Being workspace admin alone will not let you flip this setting.
2. **`EXTERNAL USE SCHEMA` privilege** on the specific schema(s) you want to mirror. This privilege was added specifically for external tools like Fabric to read UC-managed tables. Standard `SELECT` is not enough.
3. **Standard read privileges** on the catalog and schema: `USE CATALOG` on the catalog, `USE SCHEMA` and `SELECT` on the schema.

If your administrator is unsure, share the official Databricks doc: https://docs.databricks.com/en/external-access/admin.html

### From your Fabric administrator

Send them this list:

1. **Contributor role or higher on a Fabric workspace** that is backed by a Fabric capacity (F2 minimum, or a Fabric trial capacity). "Backed by capacity" means the workspace is not just My workspace or a Pro workspace.
2. **Permission to create items in that workspace** — specifically Mirrored Azure Databricks catalog items.
3. If the Databricks storage account (ADLS Gen2 backing UC) has a firewall or private endpoint, the Fabric tenant / workspace identity may need to be allowlisted. This usually only matters in enterprise setups.

---

## Part 1 — Prepare data in Databricks

### 1.1 Confirm Unity Catalog is enabled

In your Databricks workspace, click **Catalog** in the left sidebar and run this in the SQL editor:

```sql
SHOW CATALOGS;
```

You should see catalogs like `main`, `system`, and `samples` in the result. If you only see `hive_metastore` or `spark_catalog`, Unity Catalog is not attached — stop and get that fixed first (that's a separate metastore-setup exercise).

### 1.2 Create a catalog, schema, and at least one table

Simplest path — use one of Databricks' shipped sample datasets:

```sql
CREATE CATALOG IF NOT EXISTS fabric_demo;
CREATE SCHEMA IF NOT EXISTS fabric_demo.sales;
CREATE TABLE IF NOT EXISTS fabric_demo.sales.taxi_trips AS
  SELECT * FROM samples.nyctaxi.trips LIMIT 10000;
```

**Common gotcha**: if you get `Metastore storage root URL does not exist`, your metastore has no default storage location. Two fixes:

- **Easy**: create the catalog through the Catalog Explorer UI (top-right + button → Add a catalog), which uses Default Storage automatically.
- **Explicit**: specify a managed location that points at an external location you already have registered:
  ```sql
  CREATE CATALOG IF NOT EXISTS fabric_demo
    MANAGED LOCATION 'abfss://<container>@<storage-account>.dfs.core.windows.net/<path>';
  ```

Verify the table:

```sql
SELECT COUNT(*) FROM fabric_demo.sales.taxi_trips;
```

### 1.3 Enable external data access on the metastore

This is a metastore-level setting done in the workspace UI (not the account console):

1. In the Databricks workspace, click **Catalog** in the left sidebar
2. Click the **gear icon** ⚙️ at the top of the Catalog pane
3. Select **Metastore**
4. On the **Details** tab, find **External data access** and toggle it to **Enabled**

You need metastore admin rights for this. If the toggle isn't there or is grayed out, that's your permission problem.

### 1.4 Grant the required privileges

In the SQL editor:

```sql
GRANT USE CATALOG ON CATALOG fabric_demo TO `<your-email>`;
GRANT USE SCHEMA, SELECT ON SCHEMA fabric_demo.sales TO `<your-email>`;
GRANT EXTERNAL USE SCHEMA ON SCHEMA fabric_demo.sales TO `<your-email>`;
```

**Backticks around the email are required** — don't remove them. Substitute your actual login (e.g. `alice@contoso.com`).

If you'll be connecting from Fabric using an organizational account (recommended), this identity is *you*. If you'll be connecting using a service principal, grant to the service principal's application ID instead.

---

## Part 2 — Create the mirrored item in Fabric

### 2.1 Open a capacity-backed workspace

1. Go to https://app.fabric.microsoft.com and sign in with the same Entra account
2. In the left sidebar, click **Workspaces**
3. Open the workspace that runs on a Fabric capacity (not "My workspace")

If you need to create one: **+ New workspace** → give it a name → under **Advanced**, set **License mode** to your Fabric capacity → **Apply**.

### 2.2 Create the mirrored Azure Databricks catalog

1. In the workspace, click **+ New item** near the top
2. In the search box, type `Mirrored`
3. Click **Mirrored Azure Databricks catalog**

### 2.3 Set up the connection

1. Click **+ New connection**
2. Fill in:
   - **Workspace URL**: your Databricks workspace URL (e.g. `https://adb-xxxxxxxxxxxx.xx.azuredatabricks.net`). No trailing slash, no path. Get it from the Databricks resource's Overview page in the Azure portal.
   - **Connection name**: whatever helps you (e.g. `databricks-prod`)
   - **Authentication kind**: **Organizational account**
3. Click **Sign in** and complete the Entra login flow
4. Click **Connect**

**Tenant gotcha**: if the sign-in throws a "not in this tenant" error, you're signed in with an identity that doesn't exist in the workspace's tenant. Sign in with an incognito window using the tenant-specific URL, or invite the guest user to the tenant first.

### 2.4 Choose tables

1. From the catalog dropdown, pick your catalog (e.g. `fabric_demo`)
2. Expand the schema (e.g. `sales`) and check the boxes for the tables you want
3. **Keep "Automatically sync future catalog changes for the selected schema" checked** — otherwise new tables added later won't appear in Fabric automatically. You'll have to add them manually via Manage catalog.
4. Click **Next**

### 2.5 Review and create

1. Confirm the item name (defaults to the catalog name)
2. Click **Create**
3. Wait 30–60 seconds for provisioning

The workspace now shows two items: the mirrored catalog and a linked **SQL analytics endpoint** (this is what queries the data via T-SQL).

---

## Part 3 — Verify it works

1. From the workspace, open the SQL analytics endpoint (top-right of the mirrored item, or click the separate endpoint item in the workspace list)
2. In the Explorer on the left, expand until you see your schema and table
3. Right-click a table → **New SQL query**
4. Run:
   ```sql
   SELECT TOP 100 * FROM sales.taxi_trips;
   ```

**Important**: mirrored tables live under the original schema name (`sales.taxi_trips`), **not** under `dbo`. If you get `Invalid object name 'taxi_trips'`, you're missing the schema prefix.

If rows come back, mirroring is working end-to-end.

---

## Adding more tables later

Once mirroring is set up, adding tables is easy:

1. Create the new table in Databricks in the same schema you mirrored
2. Grant `EXTERNAL USE SCHEMA` on the schema (once, if not already granted — it applies to all future tables)
3. Wait 30–60 seconds; the new table appears automatically in Fabric because you left auto-sync enabled

If you want to mirror a **new schema** (e.g. you created `fabric_demo.marketing` after the fact), that doesn't sync automatically. Open the mirrored item in Fabric → **Manage catalog** at the top → check the new schema → save.

---

## What can go wrong (troubleshooting)

| Symptom | Likely cause | Fix |
|---|---|---|
| `Invalid object name 'my_table'` in the SQL endpoint | Missing schema prefix | Use `sales.my_table`, not `my_table` |
| Can't see the External data access toggle | You're workspace admin but not metastore admin | Ask your Databricks admin to enable it |
| `GRANT EXTERNAL USE SCHEMA` fails | Insufficient privileges on the schema | You need to own the schema or be granted `MANAGE` on it |
| Connection sign-in fails with "not in this tenant" | Guest / cross-tenant identity issue | Sign in with an incognito window; get the account invited to the tenant |
| Mirrored item created but tables are empty when queried | Storage firewall blocking Fabric | Ask Databricks admin to allow Fabric's outbound IPs, or (for demos) use samples data that lives in Databricks-managed storage |
| Auto-sync isn't picking up new tables | Auto-sync was unchecked, or you added a new schema | Use Manage catalog in Fabric to add missing items |
| Delayed data updates in Fabric after Databricks writes | Normal — Delta transaction log takes a moment to propagate | Usually seconds; refresh the SQL endpoint |

---

## Cost notes

- The Databricks SQL warehouse only runs when *creating* tables or querying inside Databricks. Fabric mirroring reads Delta files directly from storage and does not keep the warehouse running.
- Serverless warehouses auto-stop after idle (default 10 minutes). Safe to let them do so — mirroring keeps working.
- Fabric charges against your capacity for the SQL analytics endpoint compute when you query. No charge for the mirrored item just sitting there.
- Storage cost stays with the ADLS account backing Databricks — Fabric doesn't duplicate the data.

---

## References

- [Microsoft Fabric mirroring for Azure Databricks (official tutorial)](https://learn.microsoft.com/en-us/fabric/mirroring/azure-databricks-tutorial)
- [Enable external data access in Unity Catalog](https://docs.databricks.com/en/external-access/admin.html)
- [EXTERNAL USE SCHEMA privilege](https://docs.databricks.com/en/data-governance/unity-catalog/manage-privileges/privileges.html)
