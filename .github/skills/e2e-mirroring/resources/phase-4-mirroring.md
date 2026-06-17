# Phase 4 — Mirroring

**Goal:** Create the Fabric **connection** + **Mirrored Database** item, grant the SQL server SAMI a workspace role, then start mirroring and poll until `Running`.

## Questions

1. **Mirror the entire database or selected tables?** — builds `mirroring.json` accordingly.
2. **Confirm start** — explicit gate before `startMirroring` (kicks off the initial snapshot, which is slow).

## Stage actions

### 4a. Fabric connection (service principal auth)

```powershell
$connBody = @{
  connectivityType = "ShareableCloud"
  displayName = "<server>-AzureSQL"
  connectionDetails = @{
    type = "SQL"; creationMethod = "SQL"
    parameters = @(
      @{ dataType = "Text"; name = "server";   value = "<server>.database.windows.net" },
      @{ dataType = "Text"; name = "database"; value = "<db>" }
    )
  }
  credentialDetails = @{
    singleSignOnType = "None"; connectionEncryption = "Encrypted"; skipTestConnection = $false
    credentials = @{
      credentialType = "ServicePrincipal"
      tenantId = "<tenantId>"
      servicePrincipalClientId = "<appId>"
      servicePrincipalSecret = $spSecret   # session memory / Key Vault — never disk
    }
  }
} | ConvertTo-Json -Depth 8
$conn = Invoke-RestMethod -Method Post -Uri "https://api.fabric.microsoft.com/v1/connections" -Headers $headers -Body $connBody
```

### 4b. Mirrored Database item

```powershell
$mirroring = @{
  properties = @{
    source = @{ type = "AzureSqlDatabase"; typeProperties = @{ connection = "<connectionId>"; database = "<db>" } }
    target = @{ type = "MountedRelationalDatabase"; typeProperties = @{ defaultSchema = "<schema>"; format = "Delta" } }
  }
} | ConvertTo-Json -Depth 8
$payload = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($mirroring))
$body = @{
  displayName = "<db>-Mirrored"
  description = "Mirrored copy of Azure SQL <db>"
  definition  = @{ parts = @( @{ path = "mirroring.json"; payload = $payload; payloadType = "InlineBase64" } ) }
} | ConvertTo-Json -Depth 8
$mdb = Invoke-RestMethod -Method Post -Uri "https://api.fabric.microsoft.com/v1/workspaces/<wsId>/mirroredDatabases" -Headers $headers -Body $body
```

### 4c. Grant SAMI a workspace role + start

```powershell
# SAMI needs a workspace role (Contributor) to write to OneLake
$rb = @{ principal = @{ id = "<sqlServerSamiPrincipalId>"; type = "ServicePrincipal" }; role = "Contributor" } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri "https://api.fabric.microsoft.com/v1/workspaces/<wsId>/roleAssignments" -Headers $headers -Body $rb

# [GATE] confirm before starting
Invoke-RestMethod -Method Post -Uri "https://api.fabric.microsoft.com/v1/workspaces/<wsId>/mirroredDatabases/<mdbId>/startMirroring" -Headers $headers

# Poll Starting -> Running
do {
  Start-Sleep -Seconds 15
  $st = Invoke-RestMethod -Method Post -Uri "https://api.fabric.microsoft.com/v1/workspaces/<wsId>/mirroredDatabases/<mdbId>/getMirroringStatus" -Headers $headers
  "status: $($st.status)"
} while ($st.status -in @('Starting','Initializing'))
```

> The agent itself must not sleep-poll in a tight loop; surface status to the user and re-check on the next turn.

## Idempotency check

- Connection: list connections, filter by `displayName`.
- Mirrored DB: list `mirroredDatabases` in the workspace, filter by `displayName`.
- Status: `getMirroringStatus` returns `Running`.

## Gate

**Mandatory gate before `startMirroring`** — the initial snapshot is slow and consumes capacity.

## State written

`mirroring.connectionId`, `mirroring.connectionName`, `mirroring.itemId`, `mirroring.itemName`,
`mirroring.sqlEndpoint`, `mirroring.sqlEndpointId`, `mirroring.scope` (`FullDatabase` | table list),
`mirroring.status`.

## Verify replication

- `getMirroringStatus` → expect `Running`.
- `getTablesMirroringStatus` → per-table row counts (used by the Phase 6 lineage/table map).
