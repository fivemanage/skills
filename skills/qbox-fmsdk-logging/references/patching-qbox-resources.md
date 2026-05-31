# Patching Qbox Resources for Logging

Use this when a Qbox action cannot be observed from another resource.

## Good patch shape

Add one internal audit event after the action succeeds:

```lua
TriggerEvent('fm-qbox:server:<resource>:<action>', source, payload)
```

Example:

```lua
-- After a successful admin permission change
TriggerEvent('fm-qbox:server:qbx_adminmenu:permissionChanged', source, {
    targetSource = selectedPlayer.id,
    permission = input,
})
```

Then centralize the fmsdk call in `fm-qbox`:

```lua
AddEventHandler('fm-qbox:server:qbx_adminmenu:permissionChanged', function(playerSource, data)
    exports.fmsdk:Warn('admin', 'qbx_adminmenu.player.permissionChanged', {
        playerSource = playerSource,
        targetSource = data.targetSource,
        permission = data.permission,
    })
end)
```

## Why not direct fmsdk everywhere?

Central audit events keep Qbox resources loosely coupled:

- No `fmsdk` dependency in upstream resources.
- One config controls datasets and event categories.
- Logs can be disabled without modifying Qbox again.
- Patches are small and easy to rebase when Qbox updates.

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
TriggerEvent('fm-qbox:server:qbx_adminmenu:deniedAction', source, {
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
TriggerServerEvent('fm-qbox:server:qbx_adminmenu:clientOptionUsed', 'godmode', godmode)
```

Server listener example:

```lua
RegisterNetEvent('fm-qbox:server:qbx_adminmenu:clientOptionUsed', function(action, enabled)
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
