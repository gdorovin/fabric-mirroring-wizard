# Phase 5 — Direct Lake Semantic Model (Optional)

**Goal:** Build a Direct Lake semantic model over the mirrored DB's SQL analytics endpoint, then validate it with a live DAX query.

> **Delegate depth** to the `semantic-model-authoring` skill for TMDL authoring. This phase covers
> only the mirroring-specific wiring (endpoint discovery, Direct Lake partition shape, validation).

## Questions

1. **Build a semantic model now?** — yes/no. If no, skip to Phase 6.
2. **Which tables / schema?** — default = all mirrored user tables in the chosen schema (exclude operational/binary columns).
3. **Auto-detect relationships & add starter measures?** — yes/no.

## Key mechanics

- **Endpoint discovery:** read `mirroring.sqlEndpoint` from the state file (captured in Phase 4).
- **Definition format:** the semantic model create-with-definition requires `format: "TMDL"` in the
  definition object **plus** a `definition.pbism` part. Without `format`, the API returns `MissingDefinitionParts`.
- **Direct Lake partition:** `mode: directLake` with `entityName` / `schemaName` / `expressionSource`,
  pointing at a shared `DatabaseQuery` expression: `Sql.Database(endpoint, db)`.
- **Type mapping (SQL → TMDL):** int family → `int64`; decimal/money → `decimal`; float/real → `double`;
  date/datetime → `dateTime`; bit → `boolean`; else `string`.

## Create (REST shape)

```powershell
# Generate TMDL folder, base64-encode each part, POST to the semanticModels API.
# Each part: { path = "<relative>"; payload = <base64>; payloadType = "InlineBase64" }
# definition = @{ format = "TMDL"; parts = @( ...; <model.tmdl>; <expressions.tmdl>; definition.pbism ) }
Invoke-RestMethod -Method Post `
  -Uri "https://api.fabric.microsoft.com/v1/workspaces/<wsId>/semanticModels" `
  -Headers $headers -Body $body
```

## Validate (live DAX)

```powershell
$pt = az account get-access-token --resource "https://analysis.windows.net/powerbi/api" --query accessToken -o tsv
$h  = @{ Authorization = "Bearer $pt"; "Content-Type" = "application/json" }
$q  = @{ queries = @(@{ query = "EVALUATE ROW(""Customers"", COUNTROWS('Customer'), ""Orders"", COUNTROWS('SalesOrderHeader'))" }) } | ConvertTo-Json -Depth 6
Invoke-RestMethod -Method Post -Uri "https://api.powerbi.com/v1.0/myorg/datasets/<modelId>/executeQueries" -Headers $h -Body $q
```

## Idempotency check

- Semantic model: list `semanticModels` in the workspace, filter by `displayName`.
- A successful `executeQueries` response confirms tables, relationships, and measures resolve over Direct Lake.

## Gate

None required, but confirm the table/measure selection before creating.

## State written

`semanticModel.name`, `semanticModel.id`, `semanticModel.tableCount`, `semanticModel.relationshipCount`,
`semanticModel.measureCount`, `semanticModel.validated` (bool).
