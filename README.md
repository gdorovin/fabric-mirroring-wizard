# Fabric Mirroring Wizard

A guided, **policy-adaptive** agent that walks a user from zero to a running **Azure SQL → Microsoft Fabric mirroring** deployment — then documents the result with an architecture diagram.

The agent asks a sequence of questions, validates each answer, and runs every step as an **idempotent stage** with checkpoints before slow or irreversible actions (starting mirroring, resuming a capacity). It is safe to re-run: a state file records progress so a re-run resumes instead of redoing work.

## What it produces

By the end of a session you have:

1. An Azure SQL source database (existing or the AdventureWorksLT sample), policy-compliant.
2. A service principal + managed-identity wiring for Fabric to read the source and write to OneLake.
3. A Fabric workspace on an active capacity.
4. A **Mirrored Database** item, mirroring to `Running`.
5. *(optional)* A Direct Lake **semantic model** over the mirrored tables.
6. A **deployment record + architecture diagram** generated from the actual deployed values.

## How it works

- **Orchestrator agent:** [`.github/agents/FabricMirroringWizard.agent.md`](.github/agents/FabricMirroringWizard.agent.md) — drives the question flow and delegates depth to the skill.
- **Skill:** [`.github/skills/e2e-mirroring/SKILL.md`](.github/skills/e2e-mirroring/SKILL.md) — the 6-phase workflow, policy-adaptation logic, state-file schema, and documentation phase.

## Design principles

| Principle | Meaning |
|-----------|---------|
| **Policy-adaptive** | Never hardcodes a tenant's policy (e.g. MCAPS). Tries the standard path, reads the policy from any `RequestDisallowedByPolicy` error, and remediates. Pre-flight checks surface known blockers (auth mode, allowed regions, required tags). |
| **Idempotent** | Every create checks existence first. Safe to re-run after a failure. |
| **Resumable** | A `mirroring-config.json` state file records answers + completed stages. |
| **Gated** | Confirms before slow/irreversible steps. |
| **Secret-safe** | Service principal secrets go to Key Vault or session memory — never to disk or the state file. |
| **Self-documenting** | Final phase emits a deployment record + architecture diagram from real deployed values. |

## Prerequisites

See **[PREREQUISITES.md](PREREQUISITES.md)** for the full list. In short, you need:

- **Tooling:** Azure CLI (`az`) + the `microsoft-fabric` extension (preview), PowerShell 7+.
- **MCP servers (optional):** Azure MCP + Fabric MCP accelerate discovery; the agent falls back to `az` / `az rest` if none are configured.
- **Azure rights:** Contributor on the target subscription/RG, Entra admin on the SQL server, and the ability to create an app registration (or supply an existing service principal).
- **Fabric rights:** workspace creation enabled, Admin on the target workspace, capacity admin — and **mirroring enabled at the tenant level**.
- **Capacity:** an active F-SKU capacity (the wizard can create or resume one).
- **Policy read access:** Reader, so the wizard's pre-flight can detect tenant policies and adapt.

Run the [quick self-check](PREREQUISITES.md#quick-self-check) to confirm your baseline.

## Related skills (composed, not duplicated)

- `semantic-model-authoring` — Phase 5 depth.
- `fabric-document` / `excalidraw` — Phase 6 documentation + diagrams.
- `skills-for-fabric/common/ITEM-DEFINITIONS-CORE.md` — `mirroring.json` and TMDL definitions.
