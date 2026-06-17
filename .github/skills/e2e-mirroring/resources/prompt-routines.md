# Prompt Routines

Reusable question patterns the wizard invokes across phases. Keeping them in one place keeps the
flow consistent and the phase docs short.

## Routine: existing-or-create (+ RG + region)

Used for **resource group, SQL server, capacity, and workspace** — anything that may already exist
or be created fresh.

```
ASK: "Use an existing <thing> or create a new one?"
  ├─ Existing:
  │     LIST candidates (az ... list / REST list) → present as a numbered pick list
  │     RETURN the chosen id (never ask the user to type a GUID)
  └─ Create new:
        ASK name (validate against naming rules: length, allowed chars)
        IF the thing lives in a resource group:
            invoke this routine recursively for the resource group
        ASK region:
            IF an allowed-locations policy applies → offer only allowed regions
            ELSE default to the parent/RG region
        COLLECT any policy-required inputs (tags, SKU from allowed set)
        CONFIRM (gate) → create → RETURN new id
```

### Implementation notes

- **Always list, never demand a GUID.** Present `name` (and `state`/`region` where useful) and map the pick back to the id internally.
- **Validate names locally** before calling the API (fail fast): RG, server, capacity, and workspace each have distinct rules.
- **Recurse for RG.** Creating a server/capacity may require first creating its resource group — reuse this same routine.
- **Policy-aware region.** Pull the allowed set from the Phase 0 pre-flight (`policy.allowedRegions`) when present.

## Routine: confirm-gate

Used before slow or irreversible actions (resume capacity, start mirroring, create app registration, assign role).

```
STATE plainly what is about to happen and its consequence (cost / time / directory change).
ASK: "Proceed? (yes/no)"
  ├─ yes → execute, then record completion in the state file
  └─ no  → stop the phase; leave state unchanged; offer to revisit later
```

## Routine: pick-from-list

```
GIVEN a REST/CLI list result:
  FILTER to user-owned / relevant items
  DISPLAY as: <index>. <displayName>  [<extra: state/region/sku>]
  MAP the chosen index → object → id
```

## Principle

> One question at a time. Only ask what cannot be auto-discovered. Prefer a pick list over typed input.
