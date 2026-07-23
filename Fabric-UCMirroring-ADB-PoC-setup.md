# Fabric Mirroring for Azure Databricks Unity Catalog — Private-Network PoC

> **Audience:** Customer architects and platform engineers evaluating a proof-of-concept.
> **Assumes:** Working knowledge of Azure, Fabric, networking, and Databricks. Implementation steps and validation checkpoints only.

---

## 1. Scenario

Mirror Azure Databricks Unity Catalog tables into Fabric with **no data movement** — only the catalog structure is mirrored, and the underlying data is accessed through shortcuts — where the Databricks workspace is **private** and Fabric reaches it over a **virtual network (VNet) data gateway**.

---

## 2. Prerequisites

### 2.1 Microsoft Fabric
- Fabric capacity on an **F SKU** (required for trusted workspace access).
- A **workspace identity** created on the Fabric workspace.
- Region on the [mirroring support list](https://learn.microsoft.com/en-us/fabric/mirroring/azure-databricks-limitations).
- Fabric Runtime **≥ Spark 3.4 / Delta 2.4**.

### 2.2 Azure Databricks & Unity Catalog
- Databricks workspace with **Unity Catalog enabled**.
- **External data access enabled** on the metastore.
- Configuring identity is a Databricks **workspace user/admin** holding **`EXTERNAL USE SCHEMA`** on the target schema.
- Mirrored table data lives in **UC managed storage** or a **UC external location** (see §5.1) — not on the workspace DBFS root.

### 2.3 Networking
- Databricks deployed via **VNet injection** with **front-end (inbound) Private Link** configured.
- The **UC managed storage** and/or **external-location** ADLS Gen2 account(s) backing the Delta tables must be reachable by Fabric.

---

## 3. Implementation Steps

### Step 1 — Enable external access in Unity Catalog
**Actions**
- Enable external data access on the metastore.
- Grant **`EXTERNAL USE SCHEMA`** on the target schema.
  - Credential vending issues **short-lived credentials**, refreshed hourly and revocable in Unity Catalog.

**Docs:** [Access Databricks data using external systems](https://learn.microsoft.com/en-us/azure/databricks/external-access/) · [Mirroring tutorial](https://learn.microsoft.com/en-us/fabric/mirroring/azure-databricks-tutorial)

### Step 2 — Configure Databricks front-end Private Link
**Actions**
- Deploy via VNet injection and create the workspace private endpoint (**`databricks_ui_api`**).
  - Add **`browser_authentication`** for organizational-account SSO.
- Enforce private connectivity so the workspace **rejects all public network connections**.

**Docs:** [Azure Private Link concepts](https://learn.microsoft.com/en-us/azure/databricks/security/network/concepts/private-link) · [ADB behind a private endpoint](https://learn.microsoft.com/en-us/fabric/mirroring/azure-databricks-private-endpoint)

### Step 3 — Create the Fabric VNet data gateway
**Actions**
- Create the VNet data gateway in the **same region** as the Databricks workspace.
- Ensure the gateway can **reach the workspace private endpoint** (e.g., deploy it in the same VNet).

**Docs:** [ADB behind a private endpoint](https://learn.microsoft.com/en-us/fabric/mirroring/azure-databricks-private-endpoint)

### Step 4 — Create the Azure Databricks connection (pre-create it)
**Actions**
- Go to **Settings → Manage connections and gateways → New → Virtual network connection**.
  - Select the gateway, connection type **Azure Databricks workspace**, the workspace URL, and **organizational-account** or **service-principal** auth.
- Create it **before** the mirror item — a known issue blocks selecting the gateway during item creation.

**Docs:** [ADB behind a private endpoint](https://learn.microsoft.com/en-us/fabric/mirroring/azure-databricks-private-endpoint)

### Step 5 — *(If storage is firewalled)* Enable Trusted Workspace Access to ADLS Gen2
**Actions**
- Applies to the **UC managed storage** and **UC external-location** ADLS Gen2 account(s) that back the mirrored Delta tables (see §5.1).
- Grant the identity **Storage Blob Data Reader** (folder-level **Read/Execute** recommended).
- Add a **resource instance rule** for the Fabric workspace on the storage account. (This _is_ the firewall rule, and you can view it (not edit) in Azure Portal, Storage Account, Networking, Resource Settings, Resources instances)
  - **ARM template or PowerShell only** — the Azure portal is not supported.
  - Fabric uses the **Workspace Identity** to traverse the storage firewall, regardless of the connection's auth method.

**Docs:** [Trusted workspace access](https://learn.microsoft.com/en-us/fabric/security/security-trusted-workspace-access) · [Secure Fabric mirrored databases](https://learn.microsoft.com/en-us/fabric/mirroring/azure-databricks-security)

### Step 6 — Create the Mirrored Azure Databricks catalog item
**Actions**
- **New item → Mirrored Azure Databricks catalog**; select the existing connection.
- Choose the catalog, schemas, and tables (inclusion/exclusion list).
  - For firewalled storage, use the **Network Security** tab with a **folder-scoped endpoint**.
- **Review and Create.**

**Docs:** [Mirroring tutorial](https://learn.microsoft.com/en-us/fabric/mirroring/azure-databricks-tutorial)

---

## 4. Validation

- [ ] Mirrored item is created with **one Databricks-type shortcut per table**.
- [ ] Query the mirrored tables via the **SQL analytics endpoint** (T-SQL) to confirm data is readable.
- [ ] Shortcuts appear **only** for tables whose storage account matches the ADLS connection.
- [ ] Firewall-enabled storage connections show status **Offline** in Manage connections — **expected, not an error**.
- [ ] Add a schema/table in Unity Catalog and confirm **metadata sync** to Fabric.

---

## 5. Networking Notes

| Concern | Determination |
|---|---|
| **Public access disabled?** | **Yes** — on both the Databricks workspace (gateway reaches it privately) and the table-data ADLS Gen2 (trusted workspace access supports public access disabled). |
| **Azure Databricks Workspace Private Link** | **Required.** Front-end/inbound Private Link is a prerequisite; the VNet data gateway terminates against that workspace private endpoint. |
| **Microsoft Fabric workspace Private Link** | **Not required.** An inbound control over access *to the Fabric workspace*, independent of the outbound mirroring path. Enable only to lock down inbound access to the Fabric workspace itself. |
| **Trusted Workspace Access** | **Storage path only.** Reaches the firewalled ADLS Gen2 that backs the tables (UC managed storage and external locations — see §5.1), independent of the Databricks connection. |
| **Fabric VNet data gateway** | **Databricks path only.** Same region as the workspace; on-premises data gateway is **not** supported. |

### 5.1 Table storage behind a firewall — what Trusted Workspace Access covers

Fabric reads the underlying Delta files from whichever ADLS Gen2 account backs each mirrored table. Trusted Workspace Access makes those accounts reachable when they are firewalled — **but only for the storage that actually holds table data**:

| Databricks storage | Backs mirrored tables? | Firewalled → works via Trusted Workspace Access? |
|---|---|---|
| **UC managed storage** (managed tables & volumes) | Yes (Delta) | ✅ **Yes** |
| **UC external locations** (external tables) | Yes (Delta only) | ✅ **Yes** |
| **Workspace storage / DBFS root** | No — UC table data does not live here | ❌ **Not supported** |
| **Internal DBFS** | No | ❌ Never accessible (by design) |

- **Mechanism (for the ✅ rows):** allow the Fabric **Workspace Identity** in the Azure Storage firewall and add a **resource instance rule** for the Fabric workspace. Fabric uses the Workspace Identity to traverse the firewall regardless of the connection's auth method.
- **Not reachable:** the Databricks **workspace storage account (DBFS root)** and **internal DBFS**. Legacy `hive_metastore` managed tables live on the DBFS root and **cannot be mirrored** — migrate them into UC managed storage or an external location.
- **Constraints:** one storage account per catalog connection; shortcuts are created **only** for tables whose storage account matches the connection; **Delta format only**.

---

## 6. Important Limitations

- **Unsupported table types:** RLS/CLM tables, Lakehouse federated tables, Delta sharing tables, streaming tables, views, and materialized views. **Delta format only.**
- **Storage scope:** works with **UC managed storage** and **UC external locations**; the **workspace storage account / DBFS root behind an Azure Storage firewall is not supported**, and legacy `hive_metastore` managed tables (stored on the DBFS root) cannot be mirrored.
- **Governance is not inherited:** Unity Catalog permissions are not mirrored; the **connection creator's credential is used for all queries**, so UC access controls do not apply to Fabric users — re-implement access in Fabric.
- **Connection ordering:** the VNet-gateway connection cannot be created during mirror-item creation; **pre-create it**.
- **Other:** renaming a schema/table in the inclusion/exclusion list is not supported; trusted workspace access is not available in **Trial** capacities.

---

## 7. Sources

1. [Mirroring Azure Databricks Unity Catalog](https://learn.microsoft.com/en-us/fabric/mirroring/azure-databricks)
2. [Tutorial: Configure mirrored databases from Azure Databricks](https://learn.microsoft.com/en-us/fabric/mirroring/azure-databricks-tutorial)
3. [Mirrored Azure Databricks behind a private endpoint](https://learn.microsoft.com/en-us/fabric/mirroring/azure-databricks-private-endpoint)
4. [Secure Fabric mirrored databases from Azure Databricks](https://learn.microsoft.com/en-us/fabric/mirroring/azure-databricks-security)
5. [Limitations in mirrored databases from Azure Databricks](https://learn.microsoft.com/en-us/fabric/mirroring/azure-databricks-limitations)
6. [Trusted workspace access in Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/security/security-trusted-workspace-access)
7. [Overview of workspace-level private links (Fabric)](https://learn.microsoft.com/en-us/fabric/security/security-workspace-level-private-links-overview)
8. [Use Microsoft Fabric to read Unity Catalog data (Databricks)](https://learn.microsoft.com/en-us/azure/databricks/partners/bi/fabric-mirror)
9. [Azure Private Link concepts (Azure Databricks)](https://learn.microsoft.com/en-us/azure/databricks/security/network/concepts/private-link)
10. [Access Databricks data using external systems](https://learn.microsoft.com/en-us/azure/databricks/external-access/)
11. [Best practices for DBFS and Unity Catalog (Azure Databricks)](https://learn.microsoft.com/en-us/azure/databricks/dbfs/unity-catalog)
