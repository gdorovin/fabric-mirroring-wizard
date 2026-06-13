# Phase 0 — Authentication & Context

**Goal:** Establish a signed-in Azure context and pick the subscription, resource group, and region the deployment will use.

## Questions

1. **Signed in?** — Check `az account show`. If it fails, run `az login`.
2. **Which subscription?** — List and let the user pick by name; store the id.
3. **Which resource group + region?** — Use the [existing-or-create prompt routine](prompt-routines.md). If creating, ask region (respecting any allowed-locations policy — see [policy-adaptation.md](policy-adaptation.md)).

## Stage actions

```powershell
# 1. Confirm sign-in
az account show 2>$null | Out-Null
if ($LASTEXITCODE -ne 0) { az login | Out-Null }

# 2. Pick subscription (list for selection — never ask for a typed GUID)
az account list --query "[].{name:name, id:id, isDefault:isDefault}" -o table
az account set --subscription "<chosen-subscription-id>"

# 3. Resource group — existing or create (region honored from policy)
az group show -n "<rg>" 2>$null | Out-Null
if ($LASTEXITCODE -ne 0) {
  az group create -n "<rg>" -l "<region>" | Out-Null
}
```

## Idempotency check

- Subscription: `az account show` returns the expected id.
- Resource group: `az group show -n <rg>` succeeds.

## Gate

None (read-only + low-risk create). Confirm region choice if a policy restricts allowed locations.

## State written

`context.subscriptionId`, `context.tenantId`, `context.resourceGroup`, `context.region`, `context.signedInUser` (objectId + display name — needed later as the Entra admin / workspace admin).

```powershell
$ctx = az account show -o json | ConvertFrom-Json
# $ctx.id, $ctx.tenantId, $ctx.user.name
$me  = az ad signed-in-user show -o json | ConvertFrom-Json
# $me.id  -> objectId for Entra admin / role assignments
```

## Pre-flight (recommended)

Run the policy pre-flight from [policy-adaptation.md](policy-adaptation.md) now, against the chosen
subscription/RG scope, so later phases can adjust questions (allowed regions, required tags,
SQL auth mode) up front instead of failing mid-flow.
