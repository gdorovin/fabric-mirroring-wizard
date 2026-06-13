---
name: e2e-mirroring
description: >
  End-to-end, policy-adaptive setup of Azure SQL → Microsoft Fabric mirroring,
  driven as a guided 6-phase question flow with idempotent stages and a resumable
  state file. Covers source database provisioning (existing or AdventureWorksLT
  sample), service-principal + managed-identity wiring, Fabric workspace/capacity
  create-or-resume, Mirrored Database item creation and start, an optional Direct
  Lake semantic model, and a final documentation phase that emits a deployment
  record + architecture diagram from the actual deployed values.
  Use when the user wants to: (1) stand up Fabric mirroring for an Azure SQL DB,
  (2) productize a mirroring runbook into a guided wizard, (3) document a mirroring
  deployment. Triggers: "fabric mirroring", "mirror azure sql to fabric",
  "mirroring wizard", "set up mirrored database", "document mirroring deployment".
---

> **Update Check — ONCE PER SESSION (mandatory)**
> The first time this skill is used in a session, check for updates before proceeding.
> - Read the local `package.json` version, then compare it against the remote version
>   via `git fetch origin main --quiet && git show origin/main:package.json` (or the GitHub API).
> - If the remote version is newer, show the changelog and update instructions.
> - Skip if the check was already performed earlier in this session.

> **CRITICAL NOTES**
> 1. To find workspace details (including its ID) from a workspace name: list all workspaces, then use JMESPath filtering.
> 2. To find item details (including its ID) from workspace ID, item type, and item name: list all items of that type in that workspace, then use JMESPath filtering.
> 3. **Never hardcode a tenant's policy.** Try the standard path; if a `RequestDisallowedByPolicy` error returns, read the policy name from the error and remediate per [policy-adaptation.md](resources/policy-adaptation.md).
> 4. **Always use `encoding='utf-8'`** when reading/writing JSON, Markdown, and definition files. Windows defaults to `cp1252`, which corrupts Unicode and base64 payloads.
> 5. **Secrets never touch disk or the state file.** Store service-principal secrets in Key Vault or session memory only.

# Azure SQL → Microsoft Fabric Mirroring (Guided)

This skill turns a manual mirroring runbook into a **guided wizard**: a sequence of
validated questions across 6 phases, each running as an idempotent stage with a
confirmation gate before any slow or irreversible action. Progress is persisted to a
state file so the flow is **resumable**.

## Prerequisite Knowledge

Reference only when a phase needs a pattern not already covered here:

- [COMMON-CORE.md](../../../skills-for-fabric/common/COMMON-CORE.md) — Fabric REST patterns, auth & token audiences, JMESPath item discovery.
- [COMMON-CLI.md](../../../skills-for-fabric/common/COMMON-CLI.md) — `az rest` / `az login` recipes.
- [ITEM-DEFINITIONS-CORE.md](../../../skills-for-fabric/common/ITEM-DEFINITIONS-CORE.md) — `MirroredDatabase` (`mirroring.json`) and semantic model (TMDL) definitions.

> If the `skills-for-fabric` repo is not present alongside this one, fetch the equivalent
> patterns from the Fabric REST API docs: https://learn.microsoft.com/en-us/rest/api/fabric/articles/

## Token Audiences

| Operation | Resource / audience |
|-----------|---------------------|
| Fabric REST (workspaces, items, mirroring) | `https://api.fabric.microsoft.com` |
| Azure SQL (grant SP, read schema) | `https://database.windows.net/` |
| ARM (enable SAMI, manage capacity) | `https://management.azure.com` |
| Power BI / DAX validation | `https://analysis.windows.net/powerbi/api` |

---

## The 6 Phases

Run phases in order. Each phase doc defines its **questions**, **stage actions**,
**idempotency check**, and **gate**. Read a phase doc only when you reach that phase.

| Phase | Goal | Resource |
|------:|------|----------|
| 0 | Auth & context — subscription, resource group, region | [phase-0-auth-context.md](resources/phase-0-auth-context.md) |
| 1 | Source database — existing or AdventureWorksLT sample, firewall | [phase-1-source-db.md](resources/phase-1-source-db.md) |
| 2 | Identity & access — service principal, SAMI, grants | [phase-2-identity-access.md](resources/phase-2-identity-access.md) |
| 3 | Fabric workspace & capacity — create or resume | [phase-3-workspace-capacity.md](resources/phase-3-workspace-capacity.md) |
| 4 | Mirroring — item create, start, poll to Running | [phase-4-mirroring.md](resources/phase-4-mirroring.md) |
| 5 | *(optional)* Direct Lake semantic model | [phase-5-semantic-model.md](resources/phase-5-semantic-model.md) |
| 6 | Document & diagram the deployment | [phase-6-document-diagram.md](resources/phase-6-document-diagram.md) |

## Cross-Cutting Mechanics

| Concern | Resource |
|---------|----------|
| Reusable "existing-or-create (+ RG + region)" prompt routine | [prompt-routines.md](resources/prompt-routines.md) |
| Policy-adaptive behavior (try-then-adapt + pre-flight) | [policy-adaptation.md](resources/policy-adaptation.md) |
| State-file schema (resume + documentation source) | [state-file-schema.md](resources/state-file-schema.md) |
| Troubleshooting / known gotchas | [troubleshooting.md](resources/troubleshooting.md) |

---

## Must / Prefer / Avoid

### Must
- Run the **update check** once per session before proceeding.
- Read the **state file** first and resume from the first incomplete phase.
- Check **existence before every create** (idempotency).
- **Gate** before resuming a capacity, starting mirroring, creating app registrations, or assigning roles.
- **Adapt to tenant policy** via the try-then-adapt pattern; never hardcode MCAPS or any rule.
- Store SP **secrets in Key Vault or session memory only**.
- Produce **Phase 6 documentation** before declaring success.
- Use `encoding='utf-8'` for all file I/O.

### Prefer
- Listing options and letting the user pick over asking for typed GUIDs.
- The AdventureWorksLT sample when the user has no existing source DB (fast demo path).
- Inline Mermaid for a quick diagram; the `excalidraw` skill when an editable artifact is wanted.
- `az fabric capacity` CLI for capacity ops; `az rest` for Fabric item ops.

### Avoid
- Hardcoded subscription/RG/server/capacity/workspace IDs (they are state, not constants).
- Assuming SQL username/password auth works — many tenants deny it.
- Starting mirroring without a confirmation gate (the initial snapshot is slow).
- Writing secrets to the state file, the deployment record, or any on-disk file.
- `Set-Content -AsByteStream` for base64 payloads on Windows PowerShell 5.1 — use `[IO.File]::WriteAllBytes` (see troubleshooting).

---

## Quick Flow (orientation only — see phase docs for detail)

```
0  az login? → pick subscription → pick/create RG (+region)
1  existing source DB? → yes: pick server+db | no: create server (policy-adaptive) + AdventureWorksLT
   → firewall (client IP + Azure services)
2  pick/create SP → grant db_owner on source → enable SQL server SAMI (ARM PATCH fallback)
3  pick capacity → resume if paused | create new (RG+region+SKU) → pick/create workspace → assign capacity
4  create MirroredDatabase item → grant SAMI workspace role → [GATE] startMirroring → poll to Running
5  (optional) build Direct Lake semantic model → validate with DAX
6  generate deployment record + architecture diagram from state file
```

## Examples

**Prompt:** "Set up Fabric mirroring for my Azure SQL database, guide me through it."
**Response shape:** Read state file (none found → fresh run). Enter Phase 0: confirm `az login`,
list subscriptions for selection, then list/offer-to-create a resource group. Proceed phase by
phase, gating before the mirroring start, ending with a generated deployment record + diagram.

**Prompt:** "Re-run the mirroring setup, it failed after creating the workspace."
**Response shape:** Read state file → Phases 0–3 complete. Resume at Phase 4 (Mirroring) without
re-creating earlier resources; verify each prior resource still exists before continuing.
