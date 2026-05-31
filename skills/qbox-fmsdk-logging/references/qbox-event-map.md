# Qbox Event Map for Fivemanage Logging

This map is based on common Qbox resource patterns and should be verified against the installed server before editing. Qbox resources change over time and servers often have local modifications.

## qbx_core

### Economy

```lua
QBCore:Server:OnMoneyChange(source, moneyType, amount, actionType, reason)
```

Use for server-wide economy dashboards.

Metadata:

- `playerSource`
- `moneyType`: `cash`, `bank`, `crypto`
- `amount`
- `actionType`: `add`, `remove`, `set`
- `reason`

Useful filters:

- `reason = 'qbx_adminmenu'`
- `reason = 'paid-bills'`
- `reason = 'Radar Fine'`
- `reason = 'police-cash-seized'`
- `reason = 'police-player-robbed'`
- `reason = 'used-moneybag'`

## qbx_management

```lua
QBCore:Server:OnPlayerLoaded()
QBCore:Server:OnPlayerUnload(source)
QBCore:Server:SetDuty(source, duty)
QBCore:Server:OnJobUpdate(source, job)
QBCore:Server:OnGangUpdate(source, gang)
qbx_core:server:onJobUpdate(jobName, job)
qbx_core:server:onGangUpdate(gangName, gang)
```

Use for duty dashboards, job/gang mutation audit trails, and group definition changes.

## qbx_police

High priority:

```lua
police:server:JailPlayer(targetSrc, time)
police:server:BillPlayer(targetSrc, price)
police:server:SeizeCash(targetSrc)
police:server:RobPlayer(targetSrc)
police:server:Impound(plate, fullImpound, price, body, engine, fuel)
police:server:TakeOutImpound(plate, garage)
police:server:Radar(fine)
```

Medium priority:

```lua
police:server:SetHandcuffStatus(isHandcuffed)
police:server:EscortPlayer(escortSrc)
police:server:KidnapPlayer(kidnapedSrc)
police:server:SearchPlayer(targetSrc)
police:server:SetTracker(targetId)
police:server:SetPlayerOutVehicle(targetSrc)
police:server:PutPlayerInVehicle(targetSrc)
police:server:FlaggedPlateTriggered(radar, plate, street)
```

Evidence and alerts:

```lua
evidence:server:CreateBloodDrop(citizenid, bloodtype, coords)
evidence:server:CreateFingerDrop(coords)
evidence:server:CreateCasing(weapon, serial, coords)
evidence:server:AddBlooddropToInventory(bloodId, bloodInfo)
evidence:server:AddFingerprintToInventory(fingerId, fingerInfo)
evidence:server:AddCasingToInventory(casingId, casingInfo)
evidence:server:ClearBlooddrops(bloodDropList)
evidence:server:ClearCasings(casingList)
police:server:policeAlert(text, camId, playerSource)
```

## qbx_bankrobbery

```lua
qbx_bankrobbery:server:setBankState(bankId)
qbx_bankrobbery:server:callCops(type, bank, coords)
qbx_bankrobbery:server:recieveItem(type, bankId, lockerId)
qbx_bankrobbery:server:SetStationStatus(key, isHit)
qbx_bankrobbery:server:removeElectronicKit()
qbx_bankrobbery:server:removeBankCard(number)
qbx_bankrobbery:server:setTimeout()
qbx_bankrobbery:server:SetSmallBankTimeout(bankId)
```

Use `warn` for robbery start, alarms, and power station hits. Use `info` for loot and item consumption unless the project treats robberies as security alerts.

## qbx_adminmenu

Server events worth patching/logging directly:

```lua
qbx_admin:server:sendReply(report, message)
qbx_admin:server:deleteReport(report)
qbx_admin:server:playerOptionsGeneral(selected, selectedPlayer, input)
qbx_admin:server:playerAdministration(selected, selectedPlayer, input)
qbx_admin:server:changePlayerData(selected, selectedPlayer, input)
qbx_admin:server:giveAllWeapons(weaponType, playerID)
```

Decode `selected` indexes for dashboard-friendly action names.

`playerOptionsGeneral` action map:

| Index | Action |
|---:|---|
| 1 | `kill` |
| 2 | `revive` |
| 3 | `freezeToggle` |
| 4 | `goto` |
| 5 | `bring` |
| 6 | `sitInVehicle` |
| 7 | `setRoutingBucket` |

`playerAdministration` action map:

| Index | Action |
|---:|---|
| 1 | `kick` |
| 2 | `ban` |
| 3 | `permissionChange` |

Patch candidates:

- `/report` creation via `SendReport`
- `/admin`, `/noclip`, `/names`, `/blips`, `/setmodel`
- `/admincar`
- `qbx_admin:server:spawnVehicle` callback
- `qbx_admin:server:clothingMenu` callback
- Client-only godmode, invisibility, vehicle godmode, infinite ammo

## Other Qbox resources to inspect

These often contain valuable logs but must be inspected before adding event maps:

- `qbx_ambulancejob` / `qbx_medical`: death, revive, hospital, EMS actions
- `qbx_drugs`: processing, selling, lab actions
- `qbx_garages`: vehicle state, depot, garage transfers
- `qbx_houserobbery`, `qbx_storerobbery`, `qbx_jewelery`, `qbx_truckrobbery`: robbery lifecycle and loot
- `qbx_vehiclekeys`, `qbx_vehicles`, `qbx_vehicleshop`, `qbx_vehiclesales`: ownership, key grants, purchases
- `qbx_properties`: property ownership, stash/door access
- `qbx_customs`: vehicle modifications and payments

Do not assume event names for these resources. Search and read first.
