# fm-qbox Module Template

Use this template when creating a new passive module in an `fm-qbox` resource.

## config.lua section

```lua
Config.QbxResourceName = {
    enabled = true,
    dataset = 'default',

    events = {
        categoryOne = true,
        categoryTwo = true,
    }
}
```

## fxmanifest.lua

```lua
server_scripts {
    'server/main.lua',
    'server/modules/qbx_resource_name.lua',
}

dependency 'fmsdk'
```

## server/main.lua

```lua
local function log(dataset, level, message, metadata)
    exports.fmsdk:Log(dataset, level, message, metadata)
end

FMLog = log
```

## server/modules/qbx_resource_name.lua

```lua
if not Config.QbxResourceName.enabled then return end

local dataset = Config.QbxResourceName.dataset
local events = Config.QbxResourceName.events

local function log(level, message, metadata)
    FMLog(dataset, level, message, metadata)
end

local function getPlayerName(src)
    if not src then return nil end
    return GetPlayerName(tostring(src))
end

if events.categoryOne then
    AddEventHandler('qbx_resource:server:someEvent', function(targetSource, amount)
        local src = source
        log('info', 'qbx_resource.someEvent', {
            playerSource = src,
            playerName = getPlayerName(src),
            targetSource = targetSource,
            targetName = getPlayerName(targetSource),
            amount = amount,
        })
    end)
end

print('^2[fm-qbox]^0 qbx_resource_name logging module loaded')
```

## Notes

- Match event parameter order exactly to the target resource.
- Use `RegisterNetEvent` only for new events that clients call directly. Use `AddEventHandler` for passive listeners.
- Include `playerSource` when the action came from a player.
- Include `targetSource` for second-player actions.
- Use lowercase dot-notation messages.
- Keep categories configurable.
