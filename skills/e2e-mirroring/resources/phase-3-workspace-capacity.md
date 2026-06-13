# Phase 3 — Fabric Workspace & Capacity

**Goal:** Ensure an **active Fabric capacity** and a **workspace** bound to it. Both follow the same create-or-reuse pattern; a paused capacity is resumed.

## Questions

1. **Which capacity?**
   - *Existing* → list via `az fabric capacity list`. If its `state` is `Paused`, ask **"Resume it?"**.
   - *Create new* → use the [existing-or-create routine](prompt-routines.md):
     - Resource group — existing or create new (+ region).
     - Region for the capacity (default = RG region; honor allowed-locations policy).
     - SKU (default **F2**; show a cost note).
     - Capacity admins (default = signed-in user).
2. **Which workspace?** — existing (list) or create new and assign to the capacity.

## Stage actions

```powershell
az extension add --name microsoft-fabric --allow-preview true 2>$null

# Capacity: list / resume / create
az fabric capacity list --query "[].{name:name, state:properties.state, sku:sku.name}" -o table

# Resume if paused
az fabric capacity resume --resource-group "<rg>" --capacity-name "<capacity>"

# Create new (only on the create path)
az fabric capacity create --resource-group "<rg>" --capacity-name "<capacity>" `
  --location "<region>" --sku "F2" `
  --administration-members "<signed-in UPN or objectId>"
```

```powershell
# Workspace: create + assign to capacity (idempotent: list first, filter by displayName)
$token   = az account get-access-token --resource "https://api.fabric.microsoft.com" --query accessToken -o tsv
$headers = @{ Authorization = "Bearer $token"; "Content-Type" = "application/json" }

$ws = (Invoke-RestMethod -Method Get -Uri "https://api.fabric.microsoft.com/v1/workspaces" -Headers $headers).value |
       Where-Object displayName -eq "<workspace>"
if (-not $ws) {
  $body = @{
    displayName = "<workspace>"
    capacityId  = "<capacityId>"
    description = "Workspace for Azure SQL mirroring"
  } | ConvertTo-Json
  $ws = Invoke-RestMethod -Method Post -Uri "https://api.fabric.microsoft.com/v1/workspaces" -Headers $headers -Body $body
}
```

## Idempotency check

- Capacity: `az fabric capacity show` returns `state = Active`.
- Workspace: listing workspaces returns one matching `displayName`; capture its id.

## Gate

Confirm before **resuming** a capacity (billing resumes) and before **creating** a new capacity (cost).

## State written

`fabric.capacityName`, `fabric.capacityId`, `fabric.capacitySku`, `fabric.capacityState`,
`fabric.workspaceName`, `fabric.workspaceId`.

## Notes

- The workspace **creator is Admin by default**; re-adding via API returns a duplicate error — no action needed.
- Serverless source DBs and paused capacities both auto-suspend to save cost; the wizard resumes on demand.
