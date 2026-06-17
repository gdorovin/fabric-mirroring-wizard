# Phase 1 — Source Database

**Goal:** Have a policy-compliant Azure SQL database to mirror — either an existing one the user points to, or the AdventureWorksLT sample created fresh.

## Questions

1. **Existing source or create the sample?**
   - *Existing* → pick the logical server and database (list via `az sql server list` / `az sql db list`).
   - *Create sample* → use the [existing-or-create routine](prompt-routines.md) for the server (RG/region/name), then create AdventureWorksLT.
2. *(create path)* **Compute tier?** — default GP Serverless Gen5 2 vCores (auto-pauses to save cost).
3. **Open firewall to your client IP + Azure services?** — yes/no.

## Stage actions — create path (policy-adaptive)

> Do **not** assume SQL username/password auth is allowed. Try the standard create; if a
> `RequestDisallowedByPolicy` error returns, branch per [policy-adaptation.md](policy-adaptation.md).
> The most common remediation is **Entra-only authentication**, but the wizard must react to the
> *actual* policy reported, not assume MCAPS.

```powershell
# Preferred when the tenant allows it — otherwise the catch branch switches to Entra-only.
# Entra-only path (policy-compliant in tenants that deny SQL auth):
az sql server create -n "<server>" -g "<rg>" -l "<region>" `
  --enable-ad-only-auth --external-admin-principal-type User `
  --external-admin-name "<signed-in display name>" `
  --external-admin-sid  "<signed-in objectId>"

# AdventureWorksLT sample (GP Serverless)
az sql db create -g "<rg>" -s "<server>" -n AdventureWorksLT `
  --sample-name AdventureWorksLT -e GeneralPurpose -f Gen5 -c 2 `
  --compute-model Serverless --backup-storage-redundancy Local

# Firewall (only if user opted in)
az sql server firewall-rule create -g "<rg>" -s "<server>" `
  -n AllowAzureServices --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
$myip = (Invoke-RestMethod -Uri "https://api.ipify.org")
az sql server firewall-rule create -g "<rg>" -s "<server>" `
  -n ClientIP --start-ip-address $myip --end-ip-address $myip
```

## Idempotency check

- Server: `az sql server show -n <server> -g <rg>` succeeds.
- Database: `az sql db show -g <rg> -s <server> -n <db>` succeeds.
- Firewall rules: `az sql server firewall-rule list` contains the named rules.

## Gate

Confirm before creating a new server/database (incurs cost). No gate for pointing at an existing DB.

## State written

`source.server`, `source.database`, `source.authMode` (`EntraOnly` | `SqlAuth`), `source.region`,
`source.computeTier`, `source.firewallOpened` (bool). Never store credentials.
