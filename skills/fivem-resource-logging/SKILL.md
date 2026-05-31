---
name: fivem-resource-logging
description: FiveM fmsdk logging strategy for Lua resources - use when deciding passive listeners vs direct instrumentation, patch-based logging, datasets, metadata, fxmanifest setup, or server/client logging boundaries.
---

# Skill: FiveM Resource Logging Strategy

## Purpose

Help FiveM developers add Fivemanage logging in the least invasive correct way. Use this skill before editing when the user asks how to log a resource, build a logging integration, add audit trails, or decide whether to patch a resource.

This skill is about strategy and safe integration patterns. For exact `fmsdk` export usage, combine it with `add-fmsdk-logging`. For Qbox resources, combine it with `qbox-fmsdk-logging`.

## Core Principle

Prefer the smallest reliable integration:

1. Passive listener in a separate logging resource when the event is observable.
2. Minimal `TriggerEvent` patch when the action is not observable externally.
3. Direct `exports.fmsdk:*` calls only when the resource is intended to own its logging dependency.
4. Client-to-server logging bridge only for client-only actions, and only with server-side validation or clear `clientReported = true` metadata.

## Observability Map

| Pattern in target resource | Can another resource passively log it? | Recommended approach |
|---|---:|---|
| `RegisterNetEvent('name', function(...) ... end)` | Yes | `AddEventHandler('name', function(...) ... end)` in the logging resource. |
| `TriggerEvent('name', ...)` and `AddEventHandler('name', ...)` | Yes | `AddEventHandler('name', ...)` in the logging resource. |
| `TriggerClientEvent(...)` only | Usually no server outcome | Log before the server trigger if server-side context exists. |
| `lib.callback.register(...)` | No clean passive hook | Patch after auth/success, or log downstream state change if one exists. |
| `lib.addCommand(...)` | No clean passive hook | Patch the command callback after auth/success. |
| Exported function called by another resource | No generic passive hook | Patch the function body or wrap only when the owner intentionally supports it. |
| Direct DB mutation | No | Patch after successful query or log via a higher-level event. |
| Client-only state toggles | No reliable server truth | Add a server event bridge with permission checks or mark as client-reported. |

## Workflow

### 1. Inspect before editing

Search the target resource for:

```text
RegisterNetEvent
AddEventHandler
TriggerEvent
TriggerServerEvent
lib.callback.register
lib.addCommand
exports(
PerformHttpRequest
lib.logger
```

Read the surrounding code to identify when the action actually succeeds. Do not log before permission checks unless the goal is denied-attempt or exploit logging.

### 2. Classify actions

Group events by dashboard domain:

| Domain | Examples | Dataset |
|---|---|---|
| Economy | money add/remove/set, purchases, payouts | `economy` or `bank` |
| Inventory | item add/remove, loot, stashes, shops | `inventory` |
| Admin | kick, ban, teleport, permissions, spawned weapons | `admin` |
| Police | jail, fine, impound, cuff, evidence | `police` |
| Vehicles | spawn, impound, garage, plate, ownership | `vehicles` |
| Robbery | robbery lifecycle, alarms, loot, cooldowns | `robbery` |
| Medical | deaths, revives, hospital, wounds | `medical` |

Ask the user before changing dataset names if the codebase does not already have a logging config.

### 3. Choose the integration pattern

#### Passive logging resource

Use when the target event is observable without modifying the target resource.

```lua
AddEventHandler('some_resource:server:action', function(targetSource, amount)
    local src = source
    exports.fmsdk:Info('default', 'Resource action happened', {
        playerSource = src,
        targetSource = targetSource,
        amount = amount,
    })
end)
```

#### Minimal event patch

Use when the action is inside a command, callback, export, or local function. Emit an internal audit event after authorization and after the mutation succeeds.

```lua
-- In the target resource, after success:
TriggerEvent('fm-logs:server:resourceAction', source, {
    targetSource = targetId,
    amount = amount,
})
```

```lua
-- In the logging resource:
AddEventHandler('fm-logs:server:resourceAction', function(playerSource, data)
    exports.fmsdk:Info('default', 'Resource action happened', {
        playerSource = playerSource,
        targetSource = data.targetSource,
        amount = data.amount,
    })
end)
```

#### Direct fmsdk call

Use only if the resource should depend directly on `fmsdk`.

```lua
exports.fmsdk:Info('admin', 'Admin kicked player', {
    playerSource = source,
    targetSource = targetId,
    reason = reason,
})
```

Then add `fmsdk` to `fxmanifest.lua` dependencies.

### 4. Metadata rules

Always put dynamic values in metadata. Keep `message` static.

Use these standard fields where possible:

| Field | Meaning |
|---|---|
| `playerSource` | Actor who triggered the event. Lets fmsdk append identifiers. |
| `targetSource` | Second player affected by the event. |
| `resource` | Source resource name when logging from a shared integration resource. |
| `action` | Decoded action name when the source event uses indexes. |
| `amount`, `item`, `count`, `reason`, `coords` | Domain-specific filters. |
| `clientReported` | Set `true` when the server cannot independently verify a client-only action. |

### 5. Safety rules

- Never log secrets, tokens, passwords, full webhook URLs, or full payment details.
- Do not trust client-supplied amounts, item names, coordinates, or target IDs without server validation.
- Do not log high-volume loops unless the user explicitly wants debug telemetry.
- Do not add `exports.fmsdk` calls to client scripts.
- Preserve existing permissions and behavior.

## References

- `references/passive-vs-patched.md`
- Use `add-fmsdk-logging` for exact export syntax and SDK config.
- Use `qbox-fmsdk-logging` for Qbox/QBX resources.
