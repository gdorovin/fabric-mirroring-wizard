# Phase 2 — Identity & Access

**Goal:** Wire the two identities Fabric mirroring needs:
1. A **service principal** that Fabric uses to *read* the source database.
2. The SQL logical server's **system-assigned managed identity (SAMI)**, which mirroring uses to *write* to OneLake.

## Questions

1. **Use an existing service principal or create `fabric-mirroring-sp`?**
   - *Existing* → ask for appId (list app registrations the user owns).
   - *Create* → register the app + create a client secret.
2. **Where to store the SP secret?** — Key Vault (recommended; ask which vault, create if needed) or session memory only. **Never disk / state file.**

## Stage actions

### 2a. Service principal + secret

```powershell
# Create app registration (idempotent: check by displayName first)
$app = az ad app list --display-name "fabric-mirroring-sp" -o json | ConvertFrom-Json
if (-not $app) {
  $app = az ad app create --display-name "fabric-mirroring-sp" -o json | ConvertFrom-Json
  az ad sp create --id $app.appId | Out-Null
}
# Secret -> session variable only (or push to Key Vault)
$cred = az ad app credential reset --id $app.appId -o json | ConvertFrom-Json
$spSecret = $cred.password   # session memory; do NOT persist
# Optional: store in Key Vault
# az keyvault secret set --vault-name "<kv>" --name "fabric-mirroring-sp" --value $spSecret | Out-Null
```

### 2b. Grant the SP `db_owner` on the source DB (Entra token, no sqlcmd)

```powershell
$dbToken = az account get-access-token --resource "https://database.windows.net/" --query accessToken -o tsv
$conn = New-Object System.Data.SqlClient.SqlConnection
$conn.ConnectionString = "Server=tcp:<server>.database.windows.net,1433;Database=<db>;Encrypt=True;"
$conn.AccessToken = $dbToken
$conn.Open()
$sql = @"
IF NOT EXISTS (SELECT 1 FROM sys.database_principals WHERE name = 'fabric-mirroring-sp')
    CREATE USER [fabric-mirroring-sp] FROM EXTERNAL PROVIDER;
ALTER ROLE db_owner ADD MEMBER [fabric-mirroring-sp];
"@
$cmd = $conn.CreateCommand(); $cmd.CommandText = $sql; $cmd.ExecuteNonQuery() | Out-Null
$conn.Close()
```

### 2c. Enable the SQL server SAMI (ARM REST PATCH fallback)

> `az sql server update --identity-type SystemAssigned` may **not persist** the SAMI in some
> environments. Use a direct ARM `PATCH`, then verify `identity.principalId` is populated.

```powershell
az rest --method patch `
  --url "https://management.azure.com/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Sql/servers/<server>?api-version=2023-08-01" `
  --body '{"identity":{"type":"SystemAssigned"}}'

# Verify
az sql server show -n "<server>" -g "<rg>" --query "identity.principalId" -o tsv
```

## Idempotency check

- App: `az ad app list --display-name fabric-mirroring-sp` returns it.
- DB user: the `CREATE USER ... IF NOT EXISTS` guard makes the grant safe to re-run.
- SAMI: `az sql server show --query identity.principalId` returns a non-empty GUID.

## Gate

Confirm before **creating an app registration** (directory object) and before **resetting/creating a secret**.

## State written

`identity.servicePrincipal.appId`, `identity.servicePrincipal.displayName`,
`identity.sqlServerSamiPrincipalId`, `identity.secretLocation` (`KeyVault:<vault>/<name>` | `session`).
Never store the secret value.
