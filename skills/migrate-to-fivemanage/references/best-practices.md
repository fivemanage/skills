# Fivemanage Logging: Best Practices

Source: https://docs.fivemanage.com/fivemanage/guides/logs/best-practices

---

## 1. Use Structured Logging

Never concatenate strings or use `string.format` to embed data values into a log message. Use the `metadata` table instead — it enables filtering and querying in the Fivemanage dashboard.

The `message` should always be a short, static, human-readable label. All dynamic values belong in metadata.

**Bad:**
```lua
-- String concatenation in message
exports.fmsdk:Info("default", "Player " .. GetPlayerName(source) .. " (ID: " .. source .. ") transferred $" .. amount .. " to " .. targetName)
```

**Good:**
```lua
exports.fmsdk:Info("bank", "Player transferred money", {
    playerSource = source,
    playerName = GetPlayerName(source),
    amount = amount,
    targetName = targetName
})
```

When migrating existing logs that use string formatting, extract every interpolated value into a metadata field and replace the message with a simple static string. For example:

**Before (Discord webhook embed):**
```lua
description = "**" .. GetPlayerName(source) .. "** removed **" .. count .. "x " .. itemName .. "** from stash `" .. stashId .. "`"
```

**After:**
```lua
exports.fmsdk:Info("inventory", "Item removed from stash", {
    playerSource = source,
    item = itemName,
    count = count,
    stashId = stashId
})
```

---

## 2. Include Contextual Metadata

Always include metadata that answers: who, what, where, which resource.

Commonly useful fields:
- `playerSource` — triggers automatic identifier appending (license, steam, discord, etc.)
- `targetSource` — for actions involving a second player
- `resource` — the resource name generating the log (useful for multi-script setups)
- `coords` — vector3/table for location-relevant events
- `item`, `amount`, `reason` — domain-specific context

```lua
exports.fmsdk:Log("inventory", "warn", "Player dropped high-value item", {
    playerSource = source,
    item = itemName,
    amount = amount,
    coords = GetEntityCoords(GetPlayerPed(source))
})
```

---

## 3. Use Appropriate Log Levels

| Level | When to use |
|-------|-------------|
| `debug` | Verbose tracing during development. Disable in production via config. |
| `info` | Normal operational events: player joins, purchases, routine actions. |
| `warn` | Unusual but non-breaking events: suspicious quantities, deprecated calls, retried operations. |
| `error` | Failures that affect functionality: DB errors, failed saves, export call failures. |
| `fatal` | Critical failures that threaten server stability or data integrity. |

Use the shorthand exports to keep call sites clean:
```lua
exports.fmsdk:Info(...)
exports.fmsdk:Warn(...)
exports.fmsdk:Error(...)
```

---

## 4. Use Datasets to Organize Logs

Datasets separate logs by domain. Create named datasets in the Fivemanage dashboard and reference them consistently in code.

Recommended dataset naming conventions:
- `default` — general/catch-all server events
- `inventory` — item add/remove, stash, shop events
- `bank` / `economy` — money transfers, transactions
- `admin` — admin commands, bans, kicks
- `vehicles` — vehicle spawn, impound, modifications
- `combat` — kills, damage, weapon use

```lua
-- Good: domain-specific datasets
exports.fmsdk:Log("inventory", "info", "Item added", { ... })
exports.fmsdk:Log("admin", "warn", "Admin teleported player", { ... })

-- Avoid dumping everything into default
exports.fmsdk:Log("default", "info", "Item added", { ... }) -- harder to filter
```

---

## 5. Leverage Automatic Player Identifier Appending

When `playerSource` is included in metadata, fmsdk automatically appends all player identifiers (license, steam, discord, IP, etc.) to the log — no manual work required.

```lua
exports.fmsdk:Info("default", "Player connected", {
    playerSource = source,  -- identifiers auto-appended
    playerName = GetPlayerName(source)
})
```

You can exclude specific identifier types via `config.json`:
```json
"excludedPlayerIdentifiers": ["ip", "live"]
```

---

## 6. Log Critical Server Events

The fmsdk config can auto-log some events, but you should also instrument your own resources for:
- Player connections and disconnections
- Resource start/stop
- Economy transactions above a threshold
- Admin actions
- Exploit detections

---

## 7. Don't Log Sensitive Raw Data

Avoid logging raw passwords, tokens, or full card numbers. Log identifiers and metadata that help trace behavior without exposing sensitive data.

---

## 8. Batching

The fmsdk handles batching internally — logs are queued and sent in intervals rather than one HTTP request per call. You do not need to implement batching yourself. Just call `exports.fmsdk:Log` directly.

---

## 9. Console Logging

Enable `"console": true` in `config.json` during development to see logs in the server console. Disable or reduce verbosity in production to avoid noise.

