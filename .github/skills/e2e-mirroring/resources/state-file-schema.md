# State File Schema

A single JSON file makes the wizard **resumable** and is the **source of truth for Phase 6
documentation**. It records answers + completed stages — **never secrets**.

- **Path:** `_runs/<workspace-or-run-name>/mirroring-config.json` (created next to where the agent runs).
- **Encoding:** always `utf-8`.
- **Secrets:** the `secretLocation` field records *where* a secret lives (Key Vault ref or `session`), never the value.

## Schema

```json
{
  "schemaVersion": 1,
  "runName": "adventureworks-demo",
  "updatedUtc": "2026-06-13T10:00:00Z",
  "phasesComplete": ["0", "1", "2", "3", "4"],
  "context": {
    "subscriptionId": "",
    "tenantId": "",
    "resourceGroup": "",
    "region": "",
    "signedInUser": { "objectId": "", "displayName": "" }
  },
  "policy": {
    "sqlAuthMode": "EntraOnly",
    "allowedRegions": [],
    "requiredTags": {},
    "notes": []
  },
  "source": {
    "server": "",
    "database": "",
    "authMode": "EntraOnly",
    "region": "",
    "computeTier": "GP_S_Gen5_2",
    "firewallOpened": true
  },
  "identity": {
    "servicePrincipal": { "appId": "", "displayName": "fabric-mirroring-sp" },
    "sqlServerSamiPrincipalId": "",
    "secretLocation": "KeyVault:<vault>/fabric-mirroring-sp"
  },
  "fabric": {
    "capacityName": "",
    "capacityId": "",
    "capacitySku": "F2",
    "capacityState": "Active",
    "workspaceName": "",
    "workspaceId": ""
  },
  "mirroring": {
    "connectionId": "",
    "connectionName": "",
    "itemId": "",
    "itemName": "",
    "sqlEndpoint": "",
    "sqlEndpointId": "",
    "scope": "FullDatabase",
    "status": "Running"
  },
  "semanticModel": {
    "name": "",
    "id": "",
    "tableCount": 0,
    "relationshipCount": 0,
    "measureCount": 0,
    "validated": false
  },
  "documentation": {
    "deploymentRecordPath": "",
    "diagramPath": "",
    "generatedUtc": ""
  }
}
```

## Resume logic

1. On start, read the state file. If absent → fresh run, begin Phase 0.
2. Find the first phase **not** in `phasesComplete`; resume there.
3. Before resuming, **re-verify** each previously-completed resource still exists (existence checks
   from each phase doc). If one is missing, re-run that phase.
4. After each phase, append its number to `phasesComplete`, bump `updatedUtc`, and write the file (`utf-8`).

## Rules

- Never write a secret value into this file.
- IDs are **state**, not constants — they live here, never hardcoded in scripts or docs.
- The Phase 6 documentation reads exclusively from this file so docs reflect *actual* deployed values.
