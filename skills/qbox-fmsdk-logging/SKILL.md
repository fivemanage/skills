---
name: qbox-fmsdk-logging
description: Qbox QBX Fivemanage logging for qbx_core and resources under [qbx]/ - use when patching installed qbx_* resources, commands, callbacks, exports, and client/server actions with fmsdk logs.
---

# Skill: Qbox Fivemanage Logging

## Purpose

Help users add Fivemanage structured logging to Qbox/QBX servers and resources. Use this skill for resources named `qbx_*`, folders under `[qbx]/`, `qbx_core`, `qbx_police`, `qbx_management`, `qbx_bankrobbery`, or `qbx_adminmenu`.

This skill assumes Lua resources and the Fivemanage SDK resource named `fmsdk`.

## Qbox Logging Strategy

Use this priority order:

1. Patch the installed Qbox resource directly at the point where the action succeeds.
2. Add `fmsdk` as a dependency in that resource's `fxmanifest.lua` when adding direct `exports.fmsdk:*` calls.
3. Use framework-level Qbox events only when patching a central resource like `qbx_core` gives better coverage than many individual patches.
4. Do not create or update an `fm-qbox` wrapper/aggregator resource unless the user explicitly asks for one.

## Required Exploration

Before writing code, inspect the installed resource. Do not rely only on upstream docs.

Search for:

```text
RegisterNetEvent
AddEventHandler
TriggerEvent
TriggerServerEvent
lib.callback.register
lib.callback.await
lib.addCommand
exports.qbx_core
exports.ox_inventory
SetMoney
AddMoney
RemoveMoney
AddItem
RemoveItem
DropPlayer
SetPlayerRoutingBucket
```

Classify each action as a direct patch point, downstream-framework-covered action, or client-only action that needs a server bridge.

## Known Qbox Patch Points

### qbx_core economy

The best server-wide economy patch point is `qbx_core`'s money-change event path. Add a server-side listener in `qbx_core` or patch the money event emitter, then log via `fmsdk`:

```lua
AddEventHandler('QBCore:Server:OnMoneyChange', function(source, moneyType, amount, actionType, reason)
    local messages = {
        add = 'qbx_core.money.add',
        remove = 'qbx_core.money.remove',
        set = 'qbx_core.money.set',
    }

    exports.fmsdk:Info('economy', messages[actionType] or 'qbx_core.money.changed', {
        playerSource = source,
        moneyType = moneyType,
        amount = amount,
        actionType = actionType,
        reason = reason or 'unknown',
    })
end)
```

This captures all `player.Functions.AddMoney`, `RemoveMoney`, and `SetMoney` calls, including admin menu money edits when the reason is `qbx_adminmenu`.

### qbx_management job/gang

Useful framework events to log from a direct Qbox patch:

```lua
QBCore:Server:OnPlayerLoaded
QBCore:Server:OnPlayerUnload
QBCore:Server:SetDuty
QBCore:Server:OnJobUpdate
QBCore:Server:OnGangUpdate
qbx_core:server:onJobUpdate
qbx_core:server:onGangUpdate
```

These cover duty state, player job/gang changes, and job/gang definition updates.

### qbx_police

Common server events worth patching directly in `qbx_police` include:

```lua
police:server:JailPlayer
police:server:BillPlayer
police:server:SeizeCash
police:server:RobPlayer
police:server:Impound
police:server:TakeOutImpound
police:server:Radar
police:server:SetHandcuffStatus
police:server:EscortPlayer
police:server:KidnapPlayer
police:server:SearchPlayer
police:server:SetTracker
police:server:FlaggedPlateTriggered
police:server:policeAlert
evidence:server:CreateBloodDrop
evidence:server:CreateFingerDrop
evidence:server:CreateCasing
evidence:server:AddBlooddropToInventory
evidence:server:AddFingerprintToInventory
evidence:server:AddCasingToInventory
evidence:server:ClearBlooddrops
evidence:server:ClearCasings
```

Some police events do not include final amounts. Pair them with `QBCore:Server:OnMoneyChange` and `ox_inventory` hooks when exact economy/item outcomes are required.

### qbx_bankrobbery

Common server events worth patching directly in `qbx_bankrobbery` include:

```lua
qbx_bankrobbery:server:setBankState
qbx_bankrobbery:server:callCops
qbx_bankrobbery:server:recieveItem
qbx_bankrobbery:server:SetStationStatus
qbx_bankrobbery:server:removeElectronicKit
qbx_bankrobbery:server:removeBankCard
qbx_bankrobbery:server:setTimeout
qbx_bankrobbery:server:SetSmallBankTimeout
```

Loot events identify bank/locker but may not include exact item/amount. Use `ox_inventory` logging for exact reward items if available.

### qbx_adminmenu

Server net events worth patching directly in `qbx_adminmenu`:

```lua
qbx_admin:server:sendReply
qbx_admin:server:deleteReport
qbx_admin:server:playerOptionsGeneral
qbx_admin:server:playerAdministration
qbx_admin:server:changePlayerData
qbx_admin:server:giveAllWeapons
```

Patch-required or partially covered actions:

- `lib.addCommand`: `/report`, `/admin`, `/noclip`, `/names`, `/blips`, `/admincar`, `/setmodel`, `/vec2`, `/vec3`, `/vec4`, `/heading`
- `lib.callback.register`: `spawnVehicle`, `clothingMenu`, `getReports`, `getPlayer`, `getradiolist`
- Client-only toggles: godmode, invisibility, vehicle godmode, infinite ammo, local no-clip state
- Admin give-item through `ExecuteCommand('giveitem ...')` is better logged where the `giveitem` command is implemented

## Direct Qbox Resource Patch Pattern

When adding Qbox logging, patch the installed resource directly. Add a small local helper near the top of the server file or in a shared server logging file for that resource:

```lua
local logDataset = 'admin'

local function log(level, message, metadata)
    exports.fmsdk:Log(logDataset, level, message, metadata)
end
```

Then add log calls after authorization and after the action succeeds.

Example patch inside `qbx_adminmenu` command/callback code:

```lua
log('warn', 'qbx_adminmenu.vehicle.adminCarSaved', {
    playerSource = source,
    vehicleId = vehicleId,
    model = vehName,
    targetCitizenId = playerData.citizenid,
})
```

Patch rules:

- Patch after permission checks.
- Patch after the action succeeds.
- Keep the patch small and local to the action being audited.
- Add `fmsdk` to the patched resource's `fxmanifest.lua` dependencies.
- If logging denied attempts, use `warn` and include `denied = true` plus the failed permission.
- For client-only actions, route to the server and validate ACE permissions if possible. If not fully verifiable, include `clientReported = true`.

## Message Naming

Use lowercase dot-notation for Qbox dashboards:

```text
qbx_core.money.add
qbx_police.player.jailed
qbx_management.player.clockedIn
qbx_bankrobbery.robbery.started
qbx_adminmenu.player.banned
```

Keep messages stable and put all dynamic values in metadata.

## Verification

After editing:

1. Confirm `fmsdk` is listed in the patched resource's `fxmanifest.lua` dependencies.
2. Confirm no direct `exports.fmsdk` calls were added to client scripts.
3. Confirm log calls happen after permission checks and success checks.
4. Confirm static messages and structured metadata.
5. Read the diff carefully to ensure behavior was not changed.

## References

- `references/qbox-event-map.md`
- `references/patching-qbox-resources.md`
