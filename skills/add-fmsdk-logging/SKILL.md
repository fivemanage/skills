---
name: add-fmsdk-logging
description: Instrument a FiveM resource with fmsdk structured cloud logging — use when adding logging, telemetry, or audit trails to a FiveM Lua resource using the Fivemanage SDK.
---

# Skill: Add Fivemanage SDK Logging to a FiveM Resource

## Purpose

Instrument a FiveM resource with structured cloud logging using the Fivemanage SDK (`fmsdk`). This skill covers **logging only** — not images, video, or audio.

## Background

### fmsdk logging exports

`fmsdk` is a standalone FiveM resource that exposes server-side log exports. All logging is **server-side only** — never add log calls in client-side scripts.

**Full signature:**
```lua
exports.fmsdk:Log(datasetId, level, message, metadata)
```

**Shorthand exports (preferred when level is fixed):**
```lua
exports.fmsdk:Info(datasetId, message, metadata)
exports.fmsdk:Warn(datasetId, message, metadata)
exports.fmsdk:Error(datasetId, message, metadata)
```

- `datasetId`: string name of the dataset in the Fivemanage dashboard (e.g. `"default"`, `"bank"`, `"inventory"`)
- `level`: `"debug"`, `"info"`, `"warn"`, `"error"`, `"fatal"`
- `message`: a short, static, human-readable description — **no string concatenation or `string.format`**
- `metadata`: Lua table of key-value pairs; `playerSource` and `targetSource` are special — they trigger automatic identifier appending

**Examples:**
```lua
-- Player action with context
exports.fmsdk:Info("inventory", "Item added to inventory", {
    playerSource = source,
    item = itemName,
    count = amount
})

-- Two-player interaction
exports.fmsdk:Info("bank", "Player transferred money", {
    playerSource = source,
    targetSource = targetId,
    amount = transferAmount,
    account = "savings"
})

-- Error with no player involved
exports.fmsdk:Error("default", "Database save failed", {
    error = err,
    table = "player_data"
})

-- Fatal event
exports.fmsdk:Log("default", "fatal", "Server data corruption detected", {
    details = corruptionDetails
})
```

---

## Rules

### 1. Keep messages static

The `message` argument must always be a short, static, human-readable label. Never use `..` string concatenation or `string.format` inside the message.

**Bad:**
```lua
exports.fmsdk:Info("default", "Player " .. GetPlayerName(source) .. " bought " .. itemName .. " x" .. count)
```

**Good:**
```lua
exports.fmsdk:Info("shop", "Player purchased item", {
    playerSource = source,
    playerName = GetPlayerName(source),
    item = itemName,
    count = count
})
```

All dynamic values belong in `metadata`.

### 2. Use `playerSource` for player-triggered events

When a log is triggered by a player action, always include `playerSource = source` in metadata. The SDK will automatically append all player identifiers (license, steam, discord, etc.) — you do not need to collect them manually.

```lua
exports.fmsdk:Info("inventory", "Item removed", {
    playerSource = source,   -- identifiers auto-appended
    item = itemName,
    count = count
})
```

Use `targetSource` for any second player involved in the event.

### 3. Use appropriate log levels

| Level | When to use |
|-------|-------------|
| `debug` | Verbose tracing during development. Disable in production via config. |
| `info` | Normal operational events: player joins, purchases, routine actions. |
| `warn` | Unusual but non-breaking events: suspicious quantities, retried operations. |
| `error` | Failures that affect functionality: DB errors, failed saves, export failures. |
| `fatal` | Critical failures that threaten server stability or data integrity. |

Use the shorthand exports (`Info`, `Warn`, `Error`) to keep call sites clean. Use `Log` directly only when you need `debug` or `fatal`.

### 4. Choose a meaningful dataset

Organize logs by domain. Name datasets to match what you create in the Fivemanage dashboard.

| Dataset | Used for |
|---------|----------|
| `default` | General server events, catch-all |
| `inventory` | Item add/remove, shops, stashes |
| `bank` / `economy` | Money transfers, transactions |
| `admin` | Admin actions, bans, kicks |
| `vehicles` | Spawns, modifications, impounds |
| `combat` | Kills, damage, weapon events |

Do not dump every log into `"default"` — use domain-specific datasets to make filtering useful.

### 5. Include rich contextual metadata

Include all relevant context so that each log is self-contained and searchable.

Useful metadata fields to consider:
- `playerSource` — who triggered the event
- `targetSource` — a second player involved
- `playerName` — `GetPlayerName(source)` where useful
- `coords` — `GetEntityCoords(GetPlayerPed(source))` for location-relevant events
- `item`, `amount`, `reason` — domain-specific values
- `resource` — the resource name, useful in multi-script setups

```lua
exports.fmsdk:Log("inventory", "warn", "Suspicious item quantity detected", {
    playerSource = source,
    item = itemName,
    amount = amount,
    coords = GetEntityCoords(GetPlayerPed(source))
})
```

### 6. Log server-side only

Never add `exports.fmsdk` calls in client-side Lua files. fmsdk logging exports are server-side only. If a client event triggers something worth logging, use a server-side callback or event handler.

### 7. Don't log sensitive raw data

Never log raw passwords, tokens, full card numbers, or other sensitive values. Log identifiers and contextual metadata that support debugging without exposing sensitive data.

### 8. Don't implement batching yourself

The fmsdk handles batching internally — logs are queued and sent in intervals. Call `exports.fmsdk:Log` directly at each call site; do not buffer logs yourself.

### 9. Declare the fmsdk dependency

Add `fmsdk` as a dependency in the resource's `fxmanifest.lua` so FiveM ensures it is loaded first:

```lua
dependencies {
    'fmsdk'
}
```

---

## What to Instrument

When adding logging to a resource, focus on these categories of events:

### Player lifecycle
- Player connecting / first spawn
- Player disconnect / dropped

### Economy / transactions
- Money transfers, deposits, withdrawals
- Shop purchases
- Large or unusual transaction amounts (log as `warn`)

### Inventory
- Item added, removed, moved
- Stash access and modification
- Shop interactions

### Admin actions
- Admin teleport, kick, ban, spectate
- Spawning items or vehicles via admin menu

### Errors and failures
- Database read/write failures
- Export call failures
- Unexpected nil values or invalid state

### Exploit signals
- Client-reported values that fail server-side validation
- Repeated or suspiciously fast actions
- Values outside expected ranges

---

## Workflow

When asked to add logging to a resource:

1. **Explore** the resource's server-side files to understand the codebase:
   - Identify the domain (inventory, economy, admin, etc.)
   - List the key events and actions that occur server-side
   - Note any existing logging (print statements, ox_lib, Discord webhooks)

2. **Plan** which events to instrument based on the categories above. Prioritize:
   - Player-triggered state changes
   - Economy and inventory mutations
   - Error paths
   - Admin actions

3. **Group logs by domain and propose datasets.** Before writing any code, present a summary of the planned log groups to the user and ask them to confirm or change the dataset name for each group. Follow the rules below.

4. **Write the log calls** following all rules:
   - Static `message`
   - Dynamic values in `metadata`
   - `playerSource` for player events
   - Appropriate log level
   - Correct dataset (as confirmed by the user in step 3)

5. **Add the `fmsdk` dependency** to `fxmanifest.lua` if not already present.

6. **Do not** add logging to client-side files.

7. **Do not** remove any existing functionality — only add log calls.

8. **Inform the user** about any `server.cfg` and `config.json` setup required (see references).

---

## Dataset Confirmation Step

After planning but before writing any code, you must ask the user to confirm the dataset name(s). Follow these rules:

### Group first, ask once per group

Do not ask about every individual log. Categorize all planned logs into domain groups, assign a proposed dataset name to each group, then ask the user once per group.

For example, if the resource has item events, shop events, and error paths, that is two or three groups — not one question per log call.

### Always propose `"default"` as the fallback

Every Fivemanage account has a `"default"` dataset. If a group of logs does not clearly belong to a named domain, propose `"default"` for that group.

For domain-specific groups, propose the most appropriate name from the standard set:

| Proposed dataset | Fits when logs are about... |
|------------------|-----------------------------|
| `default` | General events, catch-all, or unclear domain |
| `inventory` | Item add/remove, stashes, shops |
| `bank` / `economy` | Money, transactions, transfers |
| `admin` | Admin commands, kicks, bans |
| `vehicles` | Spawns, impounds, modifications |
| `combat` | Kills, damage, weapon use |

### How to present the question

Show the user a clear summary. Example format:

> I found logs across **3 groups**. Here's how I plan to route them — let me know if you'd like to change any dataset name:
>
> | Group | Example events | Proposed dataset |
> |-------|---------------|-----------------|
> | Inventory events | item added, item removed, stash access | `inventory` |
> | Economy events | money transfer, shop purchase | `bank` |
> | Errors & failures | DB save failed, export error | `default` |
>
> Reply with any changes (e.g. "use `economy` instead of `bank`") or say **"looks good"** to proceed.

### After the user replies

- Apply any dataset name changes they requested
- If they approve without changes, proceed with the proposed names
- If they provide a custom name not in the standard set, use it exactly as given — do not correct or override it
- Then write all log calls

---

## fxmanifest Dependency

Always check `fxmanifest.lua` and add `fmsdk` to the `dependencies` block if it is not already there.

```lua
dependencies {
    'fmsdk'
    -- other deps...
}
```

---

## Setup Reminder (for the user)

After adding log calls, remind the user:

1. Install the `fmsdk` resource on their server (download from the [release page](https://github.com/fivemanage/sdk/releases/latest))
2. Add to `server.cfg`:
   ```
   set FIVEMANAGE_LOGS_API_KEY "your_api_key_here"
   ensure fmsdk
   ```
3. Create datasets in the Fivemanage dashboard matching the names used in the code
4. Review `config.json` inside the `fmsdk` resource (see `references/config-examples.md`)

---

## References

- `references/sdk-api.md` — fmsdk export signatures and usage examples
- `references/best-practices.md` — structured logging patterns and best practices
- `references/config-examples.md` — server.cfg and config.json setup examples
