# Policy Adaptation

The wizard **adapts to whatever the target tenant enforces** — it never hardcodes a specific policy
such as MCAPS. MCAPS is one documented case in the remediation table below, not an assumption.

## Two strategies (used together)

### 1. Try-then-adapt (primary)

Attempt the standard path. If the operation fails with `RequestDisallowedByPolicy`, parse the **policy
name** from the error and branch. This works for *any* deny policy in *any* tenant.

```powershell
try {
  az sql server create -n "<server>" -g "<rg>" -l "<region>" -u "<admin>" -p "<pwd>" 2>&1
}
catch {
  $err = $_.Exception.Message
  if ($err -match 'RequestDisallowedByPolicy') {
    # Extract the policy definition name from the error payload
    if ($err -match 'AzureADOnlyAuthentication|WithoutAzureADOnlyAuthentication') {
      # Remediation: create with Entra-only auth instead (see Phase 1)
    }
    elseif ($err -match 'AllowedLocations|allowed-locations') {
      # Remediation: re-prompt for a region from the allowed set
    }
    elseif ($err -match 'RequireTag|tag') {
      # Remediation: re-issue create with the required tags
    }
    else {
      # Unknown policy: surface the policy name + message and ask the user how to proceed
    }
  }
}
```

### 2. Pre-flight policy check (secondary)

Before acting, enumerate assignments on the target scope so questions can be adjusted up front
(e.g. only offer allowed regions, pre-fill required tags, choose Entra-only auth from the start).

```powershell
# Assignments effective at the RG/subscription scope
az policy assignment list --scope "/subscriptions/<sub>/resourceGroups/<rg>" `
  --query "[].{name:displayName, policy:policyDefinitionId}" -o table

# Non-compliant / deny states already recorded
az policy state list --resource-group "<rg>" `
  --query "[?complianceState=='NonCompliant'].{policy:policyDefinitionName, resource:resourceId}" -o table
```

## Known remediation table

| Policy pattern (any tenant) | Symptom | Remediation |
|---|---|---|
| Deny SQL local auth (MCAPS: `AzureSQL_WithoutAzureADOnlyAuthentication_Deny`) | `-u/-p` server create fails | Create server with `--enable-ad-only-auth` + Entra admin (signed-in user) |
| Allowed-locations | Region rejected | Re-prompt region from the allowed set; default capacity region to an allowed one |
| Required tags | Create rejected for missing tags | Collect required tag values as questions; pass on every create |
| Private-endpoint mandate | Public network create blocked | Surface to user; private-endpoint provisioning is out of scope — pause for manual setup |
| Allowed SKUs | Capacity/DB SKU rejected | Re-prompt SKU from the allowed set |

## Principle

> The wizard's stance is **"follow the tenant's policies, whatever they are"** — not
> "work around MCAPS." Always read the actual policy from the error or pre-flight, then adapt.
