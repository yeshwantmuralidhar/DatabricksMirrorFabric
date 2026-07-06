# Mirror Azure Databricks into Microsoft Fabric — Enterprise Setup (Service Principal)

A step-by-step guide for connecting a Databricks Unity Catalog to Microsoft Fabric in a production or shared environment, using a **service principal** for the connection identity and **trusted workspace access** to reach ADLS behind a firewall.

> **Companion document**: this is v2. See v1 (`mirror_databricks_to_fabric_guide_v1_starter.md`) for the simpler organizational-account setup useful for demos and single-user scenarios. If you're new to this, read v1 first — this document builds on it.

---

## Why do this instead of the starter setup

The v1 setup uses your personal Entra login as the connection identity. That works, but has issues for production:

| Problem with organizational account | Why it matters | How SP fixes it |
|---|---|---|
| Tied to a human | Breaks when the person leaves the org, changes roles, or has MFA issues | SP has no lifecycle tied to a person |
| Personal scope | Person likely has broader access than mirroring needs | SP gets least-privilege scope |
| Audit noise | Hard to distinguish "Fabric read data" from "Alice queried Databricks" in logs | SP actions are cleanly attributable |
| No rotation | You can't rotate a person's credentials to secure the pipeline | SP secrets can be rotated on a schedule |
| Consent surprises | Interactive sign-in may need re-consent, tenant re-selection, guest B2B setup | SP auth is non-interactive |

And ADLS-firewall setup: if your Databricks storage account has public network access disabled (which it should in an enterprise), you also need **trusted workspace access** so Fabric can read Delta files past the firewall.

---

## The identities involved (know before you start)

Enterprise mirroring involves **up to four distinct identities**. Getting them straight upfront prevents a lot of confusion later.

| # | Identity | Type | Purpose | Who configures |
|---|---|---|---|---|
| 1 | **Fabric-Databricks connection SP** | Entra app registration | The identity Fabric uses to call Unity Catalog APIs (list schemas, get storage pointers) | You — this guide |
| 2 | **Fabric workspace identity** | Fabric-managed identity | The identity Fabric uses to read Delta files from ADLS | You — enable in Fabric admin portal |
| 3 | **Databricks Access Connector** | Azure managed identity | Databricks' own identity for reading its ADLS. Not used by Fabric. | Databricks admin (already exists) |
| 4 | **Your user identity** | Entra user | Used only to set things up. Not used at runtime. | Already exists |

**Common source of confusion**: identities #1 and #2 are both "Fabric" identities but do different things. #1 talks to Unity Catalog; #2 talks to storage. Fabric handles the split internally — you configure #1 as an app registration, and #2 is enabled/named in Fabric's admin settings.

---

## Prerequisites (get these lined up first)

| Item | Why | Who provides it |
|---|---|---|
| Azure Databricks workspace with Unity Catalog | Mirroring only supports UC tables | Databricks admin |
| Fabric capacity (F2 or higher, not trial) | Trial capacities have limited SP support; enterprise needs stable capacity | Fabric admin |
| At least one Delta table in a UC catalog | The thing being mirrored | Data owner |
| Ability to create Entra app registrations | Needed for the SP | Entra admin |
| Metastore admin rights in Databricks | For the external data access setting | Databricks admin |
| Owner rights on the ADLS storage account | To configure trusted workspace access | Storage owner |
| Fabric tenant admin rights | To enable the workspace identity feature and configure workspace identity | Fabric tenant admin |

---

## Permissions to request

### From your Entra administrator

1. **Ability to create an app registration** in your Entra tenant, or have them create one for you named e.g. `sp-fabric-databricks-mirror`.
2. **Ability to create a client secret** on that app registration (or a certificate — see appendix).
3. **Note down** the app's Application (client) ID, Directory (tenant) ID, and the secret value at creation time (the secret is only shown once).

### From your Databricks administrator

1. **Add the service principal to the Databricks workspace**. In Databricks: workspace admin console → Identity and access → Service principals → Add service principal → paste the app registration's Application ID.
2. **Grant the SP metastore-level access** if you'll manage settings via it (optional — usually you flip the toggle as an admin user, not the SP).
3. **Enable external data access on the metastore** (one-time, done by a metastore admin — not the SP).
4. **Grant the SP the following privileges** on the mirrored schema:
   - `USE CATALOG` on the catalog
   - `USE SCHEMA` on the schema
   - `SELECT` on the schema (or on specific tables)
   - **`EXTERNAL USE SCHEMA`** on the schema — this is the mirroring-specific privilege
5. If you want the SP to auto-pick up newly added tables, grant on the schema level (not table level).

Docs to share: https://docs.databricks.com/en/external-access/admin.html

### From your Fabric administrator

1. **Contributor or higher** on a Fabric workspace backed by an F SKU capacity.
2. **Enable "Service principals can use Fabric APIs"** in the Fabric admin portal → Tenant settings → Developer settings. This is off by default.
3. **Add the SP to a security group** that's allowed to use Fabric APIs (the setting above is usually restricted to a specific group).
4. **Grant the SP Contributor role** on the target Fabric workspace.
5. **Enable workspace identity** for the workspace: Workspace settings → Workspace identity → Create. This creates identity #2 from the table above.
6. **Note the workspace identity's name and object ID** — you'll need these when configuring the storage firewall.

### From your Storage administrator

1. **Trusted workspace access** on the ADLS Gen2 storage account backing Databricks:
   - Networking blade → Resource instances → **+ Add** → Resource type = **Microsoft.Fabric/workspaces** → select your Fabric workspace.
   - This is the modern approach and preferred over IP allowlisting.
2. **Grant the Fabric workspace identity Storage Blob Data Reader** on the storage account (or on the specific container). This role uses identity #2 from the table.

---

## Part 1 — Create the service principal

### 1.1 Register the application in Entra

1. Go to <https://portal.azure.com> → search **Microsoft Entra ID** → open it
2. Left sidebar → **App registrations** → **+ New registration**
3. Name: `sp-fabric-databricks-mirror`
4. Supported account types: **Accounts in this organizational directory only** (single tenant)
5. Redirect URI: leave blank
6. Click **Register**
7. On the overview page, copy and save:
   - **Application (client) ID**
   - **Directory (tenant) ID**

### 1.2 Create a client secret

1. On the app's page, left sidebar → **Certificates & secrets**
2. Click **+ New client secret**
3. Description: `fabric-mirror-connection`
4. Expiry: 6 or 12 months (whatever your policy allows)
5. Click **Add**
6. **Copy the secret value immediately** — this is the only time it's shown. Store it in a keyvault. You'll paste it into Fabric later.

### 1.3 (Optional) Set up secret rotation

If your organization requires it, set up an Azure Key Vault secret with a rotation policy and reference it from your automation. Otherwise, put a calendar reminder ~30 days before expiry to rotate the secret. When the secret expires, mirroring will silently fail until you update the connection in Fabric with a new secret.

---

## Part 2 — Give the SP access in Databricks

### 2.1 Add the SP to the Databricks workspace

1. In the Databricks workspace, click your avatar (top-right) → **Settings**
2. Left sidebar → **Identity and access** → **Service principals** → **Manage**
3. Click **Add service principal**
4. Choose **Microsoft Entra ID managed** → enter the SP's Application ID (from step 1.1)
5. Give it a display name (e.g. `Fabric Mirror SP`)
6. Click **Add**
7. On the SP's profile page, ensure it has **Workspace access** entitlement. Databricks SQL access entitlement is usually not required for Fabric mirroring but doesn't hurt.

### 2.2 Enable external data access on the metastore

Same as v1 — done once per metastore. Requires metastore admin rights (not the SP; do this as a user).

1. Catalog Explorer → gear icon ⚙️ → **Metastore**
2. **Details** tab → toggle **External data access** to **Enabled**

### 2.3 Grant the SP the required privileges

In the SQL editor, run (substitute your catalog/schema/SP application ID):

```sql
-- Reference the SP by its application ID (the client ID from Entra)
GRANT USE CATALOG ON CATALOG fabric_prod
  TO `<application-id>`;

GRANT USE SCHEMA, SELECT ON SCHEMA fabric_prod.sales
  TO `<application-id>`;

GRANT EXTERNAL USE SCHEMA ON SCHEMA fabric_prod.sales
  TO `<application-id>`;
```

**Backticks required.** Use the Application (client) ID from step 1.1, not a display name.

Verify:

```sql
SHOW GRANTS ON SCHEMA fabric_prod.sales;
```

You should see the SP listed with the three privileges.

---

## Part 3 — Set up the Fabric workspace identity and firewall access

### 3.1 Enable service principal API access (tenant-level, done by Fabric admin)

1. Fabric admin portal → **Tenant settings**
2. Under **Developer settings**, find **Service principals can use Fabric APIs**
3. Enable it — restrict to a specific security group containing your SP
4. Save. Wait ~15 minutes for the setting to propagate.

### 3.2 Enable Fabric workspace identity

1. Open the target Fabric workspace
2. **Workspace settings** (top-right gear) → **Workspace identity**
3. Click **+ Workspace identity** (or **Create**)
4. This creates identity #2 — a managed identity Fabric will use to access external Azure resources on behalf of the workspace
5. Note the identity's name (usually matches the workspace name) and its object ID

### 3.3 Configure ADLS trusted workspace access

The Databricks storage account has public access disabled. We need to let this specific Fabric workspace through.

1. In Azure portal, open the storage account backing Databricks (e.g. `dbstorage...`)
2. Left sidebar → **Networking**
3. On the **Firewalls and virtual networks** section (may be labeled **Public access** on newer UIs):
   - Ensure **Enabled from selected virtual networks and IP addresses** is selected
   - Scroll to **Resource instances**
   - Click **+ Add**
   - **Resource type**: `Microsoft.Fabric/workspaces`
   - **Instance name**: select your Fabric workspace
   - Click **Save**
4. This tells the storage firewall to trust requests coming from this specific Fabric workspace's managed identity, without needing to allowlist IP ranges.

### 3.4 Grant the workspace identity storage RBAC

1. Still in the storage account, left sidebar → **Access Control (IAM)**
2. Click **+ Add** → **Add role assignment**
3. Role: **Storage Blob Data Reader** (mirroring is read-only)
4. Members: **Managed identity**
5. Managed identity type: pick the appropriate Fabric workspace identity option (may show under "Fabric" or "User-assigned managed identity" depending on how Fabric provisioned it)
6. Select your workspace identity from step 3.2
7. **Review + assign**

Wait 5–10 minutes for RBAC to propagate before testing.

---

## Part 4 — Create the mirrored item using the SP

### 4.1 Create the connection

1. In your Fabric workspace, click **+ New item**
2. Search for `Mirrored Azure Databricks catalog` → click it
3. Click **+ New connection**
4. Fill in:
   - **Workspace URL**: your Databricks workspace URL (e.g. `https://adb-xxx.xx.azuredatabricks.net`)
   - **Connection name**: `databricks-prod-sp`
   - **Authentication kind**: **Service principal**
   - **Tenant ID**: from step 1.1
   - **Service principal ID**: Application (client) ID from step 1.1
   - **Service principal key**: the secret value from step 1.2
5. Click **Connect**

If auth fails at this step:
- Verify the SP is added to the Databricks workspace with Workspace access entitlement (step 2.1)
- Verify the tenant setting "Service principals can use Fabric APIs" is enabled and has propagated (step 3.1) — this can take 15+ minutes
- Verify the secret isn't expired

### 4.2 Choose tables (same as v1)

1. Pick the catalog → expand the schema
2. Check the tables you want
3. Leave **Automatically sync future catalog changes for the selected schema** checked
4. Click **Next** → **Create**

### 4.3 Verify

Open the SQL analytics endpoint and query:

```sql
SELECT TOP 100 * FROM sales.taxi_trips;
```

Rows returning = success. If you get a storage access error at this step, the trusted workspace access or workspace identity RBAC hasn't propagated yet — wait 10 minutes and retry.

---

## Enterprise operations

### Secret rotation

The client secret you created expires. To rotate without downtime:

1. Add a new secret to the same app registration (don't delete the old one yet)
2. In Fabric, open **Manage connections and gateways** (settings → Manage connections)
3. Find your `databricks-prod-sp` connection → **Edit**
4. Paste the new secret → **Save**
5. Test by running a query through the mirrored endpoint
6. Once confirmed, delete the old secret from the app registration

### Monitoring who's reading what

Storage-side (identity #2, workspace identity):
- Enable diagnostic settings on the storage account → send to Log Analytics
- Query for reads by the workspace identity's object ID
- You'll see file-level reads from Fabric here

Databricks-side (identity #1, connection SP):
- Databricks system tables: `system.access.audit`
- Filter by `user_identity.email = '<application-id>'`
- You'll see UC metadata calls but not per-row data reads (those go through the storage identity)

Fabric-side:
- Fabric monitoring hub → the mirrored item's activity
- Capacity metrics app for compute usage on the SQL analytics endpoint

### Removing access

To decommission this mirror:

1. Delete the mirrored item in Fabric (removes the SQL endpoint too)
2. Delete the Fabric connection (removes cached credentials)
3. Revoke Databricks grants:
   ```sql
   REVOKE ALL PRIVILEGES ON SCHEMA fabric_prod.sales FROM `<application-id>`;
   REVOKE USE CATALOG ON CATALOG fabric_prod FROM `<application-id>`;
   ```
4. Remove the SP from the Databricks workspace (workspace settings → Identity and access)
5. Remove the workspace identity's Storage Blob Data Reader role
6. Remove the trusted workspace access entry from the storage account
7. Delete the Entra app registration (or disable it if there's chance you'll want to revive it)

### Multi-environment pattern

For dev / test / prod separation, most enterprises create:

- One SP per environment (`sp-fabric-databricks-mirror-dev`, `...-test`, `...-prod`)
- One Fabric workspace per environment, each with its own workspace identity
- Separate Databricks catalogs (`analytics_dev`, `analytics_prod`), each granting the appropriate SP

Don't share an SP across environments — you lose your blast-radius isolation.

---

## Troubleshooting (enterprise-specific)

| Symptom | Likely cause | Fix |
|---|---|---|
| Connection fails with "Application not authorized" | Tenant setting for SP API access not enabled or not propagated | Enable it, wait 15+ min |
| Connection fails with "Invalid client secret" | Secret expired or copied wrong | Regenerate, re-enter in Fabric |
| Connection works but tables show no data | Storage firewall blocking Fabric | Add trusted workspace access, grant workspace identity Storage Blob Data Reader |
| `EXTERNAL USE SCHEMA` grant fails | SP not added to workspace or user running grant isn't metastore admin | Add SP to workspace first, then grant as a metastore admin |
| Mirror worked, then broke after ~90 days with no changes | Client secret expired | Rotate the secret |
| Workspace identity option missing in Fabric | Tenant setting for workspace identity not enabled | Fabric admin: enable in tenant settings |
| Trusted workspace access option missing under Resource instances | Fabric integration not GA in your region, or storage account too old | Check region availability; may need to fall back to IP allowlisting |
| Everything looks right but queries return storage errors | RBAC propagation delay | Wait 10 min after adding RBAC, retry |

---

## Appendix: Certificate auth instead of secrets

Client secrets are simpler but have to be rotated. Certificate-based auth is more secure and can be paired with cert rotation via Key Vault.

Trade-offs:

| | Secret | Certificate |
|---|---|---|
| Setup complexity | Low | Higher |
| Rotation automation | Manual or KV-triggered | Full KV automation |
| Common enterprise policy | Sometimes disallowed for prod | Often required for prod |

For the certificate flow: upload a public key to the app registration → Fabric's SP auth flow needs the private key. As of writing, Fabric supports secret-based SP auth in the mirroring connection UI more reliably than certificate; check the latest docs before committing to a cert-only path.

---

## References

- [Microsoft Fabric mirroring for Azure Databricks (official)](https://learn.microsoft.com/en-us/fabric/mirroring/azure-databricks-tutorial)
- [Fabric trusted workspace access to ADLS](https://learn.microsoft.com/en-us/fabric/security/security-trusted-workspace-access)
- [Fabric workspace identity](https://learn.microsoft.com/en-us/fabric/security/workspace-identity)
- [Databricks: enable external data access](https://docs.databricks.com/en/external-access/admin.html)
- [Databricks: create and manage service principals](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/service-principals)
- [Databricks: EXTERNAL USE SCHEMA privilege](https://docs.databricks.com/en/data-governance/unity-catalog/manage-privileges/privileges.html)
