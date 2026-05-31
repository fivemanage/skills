---
name: qbox-fmsdk-logging
description: Qbox QBX Fivemanage logging for qbx_core and resources under [qbx]/ - use when adding fm-qbox modules, auditing qbx_* events, or patching Qbox commands/callbacks that cannot be passively listened to.
---

# Skill: Qbox Fivemanage Logging

## Purpose

Help users add Fivemanage structured logging to Qbox/QBX servers and resources. Use this skill for resources named `qbx_*`, folders under `[qbx]/`, `qbx_core`, `qbx_police`, `qbx_management`, `qbx_bankrobbery`, `qbx_adminmenu`, or an `fm-qbox` logging integration resource.

This skill assumes Lua resources and the Fivemanage SDK resource named `fmsdk`.

## Qbox Logging Strategy

Use this priority order:

1. Prefer passive listeners in `fm-qbox` when Qbox already emits a server event.
2. Use framework-level Qbox events when they provide better coverage than resource-specific events.
3. Add tiny upstream patches only for commands, callbacks, exports, or client-only actions that cannot be observed externally.
4. Keep upstream patches as `TriggerEvent(...)` audit hooks where possible, and keep actual fmsdk calls centralized in `fm-qbox`.

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

Classify each action as passive, downstream-framework observable, or patch-required.

## Known Passive Qbox Hooks

### qbx_core economy

The best server-wide economy hook is:

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

Useful passive hooks:

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

Common passive events include:

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

Common passive events include:

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

Passive server net events worth logging:

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

## fm-qbox Module Pattern

Use one server module per Qbox resource:

```text
fm-qbox/
├── fxmanifest.lua
├── config.lua
└── server/modules/
    ├── qbx_core.lua
    ├── qbx_police.lua
    ├── qbx_management.lua
    ├── qbx_bankrobbery.lua
    └── qbx_adminmenu.lua
```

Use a config section per module:

```lua
Config.QbxAdminMenu = {
    enabled = true,
    dataset = 'admin',
    events = {
        reports = true,
        playerActions = true,
        administration = true,
        playerData = true,
        weapons = true,
    }
}
```

In modules, use a local helper:

```lua
local dataset = Config.QbxAdminMenu.dataset

local function log(level, message, metadata)
    exports.fmsdk:Log(dataset, level, message, metadata)
end
```

## Patching Resources That Cannot Be Listened To

When passive listening is impossible, add a tiny server-side audit event after authorization and success.

Example patch inside `qbx_adminmenu` command/callback code:

```lua
TriggerEvent('fm-qbox:server:qbx_adminmenu:adminCarSaved', source, {
    vehicleId = vehicleId,
    model = vehName,
    targetCitizenId = playerData.citizenid,
})
```

Then listen in `fm-qbox`:

```lua
AddEventHandler('fm-qbox:server:qbx_adminmenu:adminCarSaved', function(playerSource, data)
    log('warn', 'qbx_adminmenu.vehicle.adminCarSaved', {
        playerSource = playerSource,
        vehicleId = data.vehicleId,
        model = data.model,
        targetCitizenId = data.targetCitizenId,
    })
end)
```

Patch rules:

- Patch after permission checks.
- Patch after the action succeeds.
- Keep the patch to one `TriggerEvent` where possible.
- Do not add `fmsdk` as an upstream dependency if central `fm-qbox` can listen to the audit event.
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

1. Confirm the module is listed in `fxmanifest.lua`.
2. Confirm the config section name matches the module code.
3. Confirm no `exports.fmsdk` calls were added to client scripts.
4. Confirm passive listeners use the same event parameter order as the target resource.
5. If patching upstream Qbox resources, read the diff carefully to ensure behavior was not changed.

## References

- `references/qbox-event-map.md`
- `references/patching-qbox-resources.md`
- `references/fm-qbox-module-template.md`
