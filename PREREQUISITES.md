# Prerequisites

Everything required to run the Fabric Mirroring Wizard end-to-end. Verify each section before
starting a run — the wizard also checks most of these at the start of each phase and fails fast
with a clear message if something is missing.

---

## 1. Tooling (local machine)

| Tool | Why | Install / check |
|------|-----|-----------------|
| **Azure CLI (`az`)** 2.60+ | All Azure + Fabric REST calls run through `az` / `az rest`. | `az version` → upgrade with `az upgrade` |
| **`microsoft-fabric` CLI extension** (preview) | Capacity list/resume/create. | `az extension add --name microsoft-fabric --allow-preview true` |
| **PowerShell 7+** (recommended) or Windows PowerShell 5.1 | Runs the snippets. 5.1 works but has the gotchas in [troubleshooting.md](.github/skills/e2e-mirroring/resources/troubleshooting.md) (byte writes, `-UseBasicParsing`, 8.3 paths). | `pwsh --version` |
| **.NET `System.Data.SqlClient`** | Used to grant the SP `db_owner` via an Entra token (no `sqlcmd` needed). | Ships with .NET / PowerShell on Windows. |
| **Git** | Update-check pattern (`git fetch origin main`). | `git --version` |

> No ODBC/JDBC driver setup is required — the wizard authenticates to Azure SQL with an Entra access token.

---

## 2. MCP servers

The wizard runs primarily on the **Azure CLI**, so MCP servers are **optional accelerators**, not hard
requirements. Enable the ones matching how you want the agent to work:

| MCP server | Used for | Required? |
|------------|----------|-----------|
| **Azure MCP** (`mcp_azure_mcp_ser_*`) | Resource groups, role assignments, SQL, Key Vault, policy queries, subscription/region discovery. | Recommended |
| **Fabric MCP** (`mcp_fabric_mcp_se_*`) | Workspace/item discovery, OneLake inspection, item-definition docs, best practices. | Recommended |
| **Microsoft Learn MCP** (`mcp_microsoft_lea_*`) | Pull authoritative Fabric REST / mirroring docs when a pattern isn't in the skill. | Optional |
| **Power BI / semantic-model MCP** (`mcp_powerbi-model_*`) | Phase 5 semantic-model authoring + DAX validation depth. | Optional (Phase 5 only) |

If no MCP servers are configured, the agent falls back to `az` / `az rest` for everything — that path
is fully documented in the phase resource files.

---

## 3. Azure permissions

The identity **running the wizard** (your signed-in `az login` user) needs:

| Capability | Role / permission | Scope |
|------------|-------------------|-------|
| Create resource groups & SQL resources | **Contributor** | Target subscription or resource group |
| Enable the SQL server managed identity (ARM PATCH) | Contributor (write on `Microsoft.Sql/servers`) | SQL server |
| Set yourself as SQL Entra admin / grant the SP `db_owner` | **Microsoft Entra admin** on the logical server | SQL server |
| **Create an app registration** + client secret (the `fabric-mirroring-sp` service principal) | **Application Developer** Entra role (or an existing SP supplied to skip this) | Microsoft Entra tenant |
| Store the SP secret | **Key Vault Secrets Officer** | Target Key Vault (if used) |
| Read policy assignments (pre-flight) | **Reader** (policy read) | Subscription / RG |

> If you **cannot** create app registrations, supply an existing service principal in Phase 2 — the
> wizard will skip creation and only grant it `db_owner`.

### Service principal (`fabric-mirroring-sp`) — what it needs
- `db_owner` on the **source** Azure SQL database (granted by the wizard in Phase 2).
- It is the credential stored in the **Fabric connection** so mirroring can read the source.

### SQL server system-assigned managed identity (SAMI) — what it needs
- Enabled on the logical server (Phase 2).
- A **Fabric workspace role** (`Contributor`) so mirroring can write Delta files to OneLake (Phase 4).

---

## 4. Fabric permissions

| Capability | Role | Where |
|------------|------|-------|
| Create a workspace | **Fabric workspace creation** enabled for your account (tenant setting) | Fabric admin portal |
| Assign roles in the workspace (to the SAMI) | **Admin** on the workspace (the creator is Admin by default) | Target workspace |
| Create Mirrored Database, connection, semantic model items | **Contributor+** | Target workspace |
| Use / resume a capacity | **Capacity admin** or contributor on the capacity | Fabric capacity |

> **Mirroring must be enabled at the tenant level.** A Fabric tenant admin enables
> *"Create Mirrored databases"* (and, if used, service-principal API access) in the admin portal.

---

## 5. Capacity & cost

- An **active Fabric capacity** (F-SKU; **F2** is the wizard default). The wizard can **create** one or
  **resume** a paused one — both gated because billing resumes.
- The **AdventureWorksLT** sample source DB defaults to **GP Serverless** (auto-pauses when idle).
- Both the serverless DB and a paused capacity **auto-suspend** to limit cost; first use resumes them.

---

## 6. Tenant policy readiness

The wizard is **policy-adaptive** (see [policy-adaptation.md](.github/skills/e2e-mirroring/resources/policy-adaptation.md)) and reacts to
whatever the target tenant enforces — it does **not** assume MCAPS. Be aware your tenant may:

- **Deny SQL local auth** → the wizard switches to **Entra-only authentication** automatically.
- **Restrict regions** (allowed-locations) → it offers only permitted regions.
- **Require tags** → it collects required tag values as questions.
- **Mandate private endpoints** → out of scope; the wizard pauses for manual networking setup.

You need enough rights to **read policy assignments** for the pre-flight check (Reader is enough).

---

## 7. Network / firewall

- The machine running the wizard needs outbound HTTPS to: `*.fabric.microsoft.com`,
  `management.azure.com`, `database.windows.net`, `login.microsoftonline.com`, `api.powerbi.com`.
- The Azure SQL server firewall must allow **Azure services** and your **client IP** (the wizard can add
  both in Phase 1) — unless a private-endpoint policy applies.

---

## Quick self-check

```powershell
az version                                   # CLI present
az extension show --name microsoft-fabric    # Fabric extension installed
az account show                              # signed in
az account get-access-token --resource https://api.fabric.microsoft.com  # Fabric token works
az ad signed-in-user show --query id -o tsv  # your objectId (Entra admin / role assignments)
```

If all five succeed, you have the baseline to start a run.
