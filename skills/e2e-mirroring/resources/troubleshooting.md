# Troubleshooting / Known Gotchas

Symptoms encountered building real Azure SQL → Fabric mirroring deployments, with fixes.

| Symptom | Cause | Fix |
|---|---|---|
| `RequestDisallowedByPolicy` on SQL server create with `-u/-p` | Tenant denies SQL local auth (e.g. MCAPS `AzureSQL_WithoutAzureADOnlyAuthentication_Deny`) | Create with `--enable-ad-only-auth` + set signed-in user as Entra admin. See [policy-adaptation.md](policy-adaptation.md) — don't assume it's MCAPS; read the actual policy. |
| SAMI `principalId` empty after `az sql server update --identity-type SystemAssigned` | The CLI update does not persist the SAMI in some environments | Use a direct ARM REST `PATCH` with `{"identity":{"type":"SystemAssigned"}}`, then verify `identity.principalId`. |
| Mirroring never leaves `Starting` | SAMI lacks a workspace role, or source SP grant missing | Grant SAMI `Contributor` on the workspace; confirm the SP is `db_owner` on the source DB. |
| `MissingDefinitionParts` creating a semantic model | Definition object lacks `format: "TMDL"` or the `definition.pbism` part | Include `format: "TMDL"` and a `definition.pbism` part in the definition. |
| `az fabric capacity ...` not found | `microsoft-fabric` extension is preview-only | `az extension add --name microsoft-fabric --allow-preview true`. |
| Capacity operations fail with "paused" | Capacity is suspended | `az fabric capacity resume` (gate first — billing resumes). |
| Workspace role assignment returns duplicate error for the creator | Creator is Admin by default | No action — ignore the duplicate. |
| Base64 payload corrupted on write | `Set-Content -AsByteStream` mangles bytes on Windows PowerShell 5.1 | Use `[IO.File]::WriteAllBytes($path, $bytes)`. |
| `Invoke-WebRequest` blocks on an IE-engine security prompt | Default parsing engine | Add `-UseBasicParsing`. |
| Relative part-path math breaks under `$env:TEMP` | `$env:TEMP` resolves to an 8.3 short path | Compute relative paths with `Resolve-Path -Relative`. |
| Unicode corrupted in JSON/MD files | Windows default `cp1252` encoding | Always read/write with `utf-8`. |
| Source DB unreachable intermittently | Serverless DB auto-paused when idle | Expected; the first connection resumes it. Allow a retry. |

## Polling guidance

- Mirroring `getMirroringStatus` progresses `Starting`/`Initializing` → `Running` after the initial snapshot.
- The **agent must not tight-loop sleep**. Surface status to the user and re-check on the next turn.
- Use `getTablesMirroringStatus` for per-table row counts (feeds the Phase 6 lineage/table map).

## Permissions to check up front (fail fast)

- Create app registrations (Phase 2).
- Assign Fabric workspace roles (Phases 3–4).
- Manage / resume the target capacity (Phase 3).
- If any is missing, stop with a clear message rather than failing mid-flow.
