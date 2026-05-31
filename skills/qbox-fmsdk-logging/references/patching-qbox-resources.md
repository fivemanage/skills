# Patching Qbox Resources for Logging

Use this when a Qbox action cannot be observed from another resource.

## Good patch shape

Add one direct `fmsdk` log call after the action succeeds:

```lua
exports.fmsdk:Warn('admin', 'qbx_adminmenu.player.permissionChanged', {
    playerSource = source,
    targetSource = selectedPlayer.id,
    permission = input,
})
```

If the file needs multiple logs, add a tiny local helper near the top of the server file:

```lua
local logDataset = 'admin'

local function log(level, message, metadata)
    exports.fmsdk:Log(logDataset, level, message, metadata)
end
```

Then use it at the action site:

```lua
log('warn', 'qbx_adminmenu.player.permissionChanged', {
    playerSource = source,
    targetSource = selectedPlayer.id,
    permission = input,
})
```

## Dependency requirement

When patching an existing Qbox resource with direct fmsdk calls, add `fmsdk` to that resource's `fxmanifest.lua` dependencies:

```lua
dependencies {
    'fmsdk'
}
```

If the resource already has `dependencies`, add `fmsdk` to the existing table instead of creating a duplicate block.

## Permission and success placement

Patch after checks like:

```lua
if not IsPlayerAceAllowed(source, perm) then return end
if not exports.qbx_core:IsOptin(source) then return end
```

Patch after success checks like:

```lua
if not player.Functions.RemoveMoney(...) then return end
if not exports.ox_inventory:AddItem(...) then return end
if not success then return end
```

If logging failed attempts is required, add a separate denied-attempt event:

```lua
exports.fmsdk:Warn('admin', 'qbx_adminmenu.action.denied', {
    playerSource = source,
    action = 'ban',
    permission = config.eventPerms.ban,
    denied = true,
})
```

## Client-only actions

Qbox admin menus often contain client-only toggles. These cannot be trusted as server truth.

Options:

1. Leave unlogged if the action has no persistent server impact.
2. Patch the client to notify the server and mark the log `clientReported = true`.
3. Move the action behind a server-authorized flow if accurate audit logs are required.

Client patch example:

```lua
TriggerServerEvent('qbx_admin:server:logClientOptionUsed', 'godmode', godmode)
```

Server listener example:

```lua
RegisterNetEvent('qbx_admin:server:logClientOptionUsed', function(action, enabled)
    local src = source
    if not IsPlayerAceAllowed(src, 'admin') then return end

    exports.fmsdk:Warn('admin', 'qbx_adminmenu.clientOption.used', {
        playerSource = src,
        action = action,
        enabled = enabled,
        clientReported = true,
    })
end)
```

## Metadata for patches

Use stable keys:

| Key | Use |
|---|---|
| `playerSource` | Admin/player actor. |
| `targetSource` | Affected player. |
| `action` | Human-readable action name. |
| `selected` | Raw menu index when useful for debugging. |
| `input` | Sanitized input. Avoid secrets and long blobs. |
| `reason` | Ban/kick/report reason. |
| `durationSeconds` | Ban duration. |
| `resource` | Original Qbox resource. |
| `clientReported` | Client-only report marker. |

## Do not patch this way

Avoid:

```lua
exports.fmsdk:Info('admin', 'Admin ' .. GetPlayerName(source) .. ' banned ' .. GetPlayerName(target), {})
```

Use:

```lua
exports.fmsdk:Warn('admin', 'qbx_adminmenu.player.banned', {
    playerSource = source,
    targetSource = target,
    reason = reason,
})
```

Avoid modifying logic order, return values, permission checks, or database writes while adding logs.
