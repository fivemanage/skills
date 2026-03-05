# Skill: Migrate Logging to fmsdk

## Purpose

Help FiveM developers migrate from legacy logging patterns — `ox_lib`'s `lib.logger` and Discord webhook logging — to the Fivemanage SDK (`fmsdk`) for structured cloud logging. This skill covers **logging only** — not images or video.

## Background

### Sources to migrate

This skill handles migration from two legacy logging patterns:

1. **`lib.logger`** — ox_lib's built-in logger
2. **Discord webhook logging** — direct HTTP POST to Discord webhooks (e.g. via `PerformHttpRequest`)

Both should be replaced with `exports.fmsdk:Log`.

---

### ox_lib logger (old pattern)

`lib.logger` is a server-side function with this signature:

```lua
lib.logger(source, event, message, ...varargs)
```

- `source`: player id, identifier string, or `-1` for server-side
- `event`: name/label for the log event
- `message`: log content
- `...varargs`: extra string tags passed as `key:value` pairs

**Example (ox_lib):**
```lua
lib.logger(source, 'BankTransfer', 'Player transferred money', 'amount:5000', 'account:savings')
lib.logger(-1, 'CreateVehicle', json.encode(vehicle))
```

ox_lib logger is configured via convars:
```
set ox:logger "fivemanage"
set fivemanage:key "YOUR_API_KEY"
```

---

### Discord webhook logging (old pattern)

Many resources send logs directly to a Discord channel via webhook. Common patterns to detect:

```lua
-- Pattern 1: PerformHttpRequest to a discord.com/api/webhooks URL
PerformHttpRequest("https://discord.com/api/webhooks/...", function() end, "POST",
    json.encode({ embeds = { { title = "...", description = "..." } } }),
    { ["Content-Type"] = "application/json" }
)

-- Pattern 2: A helper wrapper like SendToDiscord / SendDiscordLog / DiscordLog / webhook
SendDiscordLog("BankTransfer", "Player " .. GetPlayerName(source) .. " transferred $5000")

-- Pattern 3: ox_inventory / qb-core style webhook util
local webhook = GetConvar("inventory:webhook", "")
PerformHttpRequest(webhook, ...)
```

Look for:
- `PerformHttpRequest` calls targeting `discord.com/api/webhooks`
- Local webhook helper functions named things like `SendToDiscord`, `DiscordLog`, `sendLog`, `webhook`
- ConVars holding Discord webhook URLs (e.g. `inventory:webhook`, `banking:webhook`)
- `json.encode` calls building Discord embed payloads

---

### fmsdk logger (new pattern)

`fmsdk` is a standalone FiveM resource. Its log export signature:

```lua
exports.fmsdk:Log(datasetId, level, message, metadata)
-- Shorthands:
exports.fmsdk:Info(datasetId, message, metadata)
exports.fmsdk:Warn(datasetId, message, metadata)
exports.fmsdk:Error(datasetId, message, metadata)
```

- `datasetId`: string name of the dataset in the Fivemanage dashboard (e.g. `"default"`, `"bank"`, `"inventory"`)
- `level`: `"debug"`, `"info"`, `"warn"`, `"error"`, `"fatal"`
- `message`: human-readable log message
- `metadata`: Lua table with any key-value pairs; `playerSource` and `targetSource` are special — they trigger automatic identifier appending

**Example (fmsdk):**
```lua
exports.fmsdk:Log("bank", "info", "Player transferred money", {
    playerSource = source,
    amount = 5000,
    account = "savings"
})

exports.fmsdk:Error("default", "Failed to create vehicle", {
    vehicleModel = "sultanrs"
})
```

---

## Migration Rules

### 1. Replace the call signature

| ox_lib | fmsdk |
|--------|-------|
| `lib.logger(source, event, message, ...)` | `exports.fmsdk:Log(dataset, level, message, metadata)` |

### 2. Map the `source` argument

- `source` (player id) → `metadata.playerSource = source`
- `-1` (server) → omit `playerSource` or set it to a descriptive string

### 3. Map the `event` argument

The `event` string in ox_lib is a label. In fmsdk it becomes part of the message or metadata:
- Prefer: fold it into the `message` string if it's descriptive
- Or: add `event = "EventName"` to the metadata table

### 4. Map varargs tags to metadata fields

ox_lib varargs are `"key:value"` strings. Convert each to a real key-value pair in the metadata table.

**Before:**
```lua
lib.logger(source, 'BankTransfer', 'Player transferred money', 'amount:5000', 'account:savings')
```

**After:**
```lua
exports.fmsdk:Info("bank", "Player transferred money", {
    playerSource = source,
    amount = 5000,
    account = "savings"
})
```

### 5. Decompose string-formatted messages into metadata

When an existing log message uses string formatting or concatenation to embed data values directly into the message string, extract those values into metadata fields and simplify the message to a plain static description.

This applies to both ox_lib and Discord webhook logs.

**Before (ox_lib):**
```lua
lib.logger(source, 'BankTransfer', 'Player ' .. GetPlayerName(source) .. ' (ID: ' .. source .. ') transferred $' .. amount .. ' from ' .. fromAccount .. ' to ' .. toAccount)
```

**Before (Discord webhook):**
```lua
SendDiscordLog("Player " .. GetPlayerName(source) .. " removed item " .. itemName .. " x" .. count .. " from stash " .. stashId)
```

**After (both):**
```lua
exports.fmsdk:Info("bank", "Player transferred money", {
    playerSource = source,
    playerName = GetPlayerName(source),
    amount = amount,
    fromAccount = fromAccount,
    toAccount = toAccount
})

exports.fmsdk:Info("inventory", "Item removed from stash", {
    playerSource = source,
    item = itemName,
    count = count,
    stashId = stashId
})
```

**Rule:** The `message` should be a short, static, human-readable label. All dynamic values belong in `metadata`. Never use `..` string concatenation or `string.format` inside the message argument of an fmsdk log call.

### 6. Migrate Discord webhook logging

Replace Discord webhook calls and helper functions with fmsdk exports.

**Step 1 — Find the patterns:**
- Search for `PerformHttpRequest` with `discord.com` in the URL
- Search for local helpers: `SendToDiscord`, `DiscordLog`, `sendLog`, `webhook(`, etc.
- Search for ConVars like `inventory:webhook`, `banking:webhook`, `logs:webhook`

**Step 2 — Replace the call site:**

**Before:**
```lua
PerformHttpRequest("https://discord.com/api/webhooks/123/abc", function() end, "POST",
    json.encode({
        embeds = {{
            title = "Bank Transfer",
            description = "**" .. GetPlayerName(source) .. "** transferred **$" .. amount .. "** to " .. targetName,
            color = 3066993
        }}
    }),
    { ["Content-Type"] = "application/json" }
)
```

**After:**
```lua
exports.fmsdk:Info("bank", "Player transferred money", {
    playerSource = source,
    targetName = targetName,
    amount = amount
})
```

**Step 3 — Remove the webhook helper function entirely** if it exists (e.g. a local `SendToDiscord` or `DiscordLog` function). Do not keep it as a no-op.

**Step 4 — Remove webhook ConVars from server.cfg:**
```
# Remove lines like:
set inventory:webhook "https://discord.com/api/webhooks/..."
set banking:webhook "https://discord.com/api/webhooks/..."
```

### 7. Choose a dataset (ask the user)

Before writing any code, propose dataset assignments to the user and get confirmation. Do not silently pick datasets and migrate — the user must approve the mapping.

**Step 1 — Auto-categorize the logs you found.**

Examine every log call site's context: the event name, message content, surrounding code, file name, and resource name. Use these signals to propose a dataset for each logical group:

| Signal | Suggested dataset |
|--------|------------------|
| `transfer`, `deposit`, `withdraw`, `balance`, `money`, `cash`, `wage`, `salary`, `invoice` | `bank` |
| `addItem`, `removeItem`, `item`, `inventory`, `stash`, `shop`, `loot`, `weapon` | `inventory` |
| `ban`, `kick`, `warn`, `admin`, `staff`, `permission`, `setJob`, `setGroup` | `admin` |
| `spawn`, `vehicle`, `impound`, `garage`, `plate` | `vehicles` |
| `kill`, `damage`, `combat`, `shot`, `arrest` | `combat` |
| `connect`, `disconnect`, `join`, `drop`, `playerJoin` | `default` |
| Anything that doesn't fit a clear domain | `default` |

Group calls by their proposed dataset — do not list every individual call, list the groups.

**Step 2 — Present the proposed mapping and ask.**

Show the user a summary like:

> I found 14 log calls across 3 files. Here's how I'd group them into datasets:
>
> - **`inventory`** (8 calls) — item add/remove, stash access, shop purchases
> - **`admin`** (3 calls) — ban, kick, setJob events
> - **`default`** (3 calls) — player connect/disconnect, resource start
>
> Does this look right? You can rename any dataset or move calls to a different one. The `default` dataset always exists — any name you use will be created automatically when the first log is sent.

**Step 3 — Wait for the user's response before writing any code.**

Accept changes like:
- "Use `economy` instead of `bank`"
- "Put everything in `default`"
- "Rename `default` to `general`"
- "The admin logs should also go to `default`"

Apply the user's corrections, then proceed with the migration using the confirmed mapping.

### 8. Choose an appropriate log level

ox_lib has no concept of log levels — everything is one level. Discord embeds sometimes use color to imply severity. When migrating, assign levels based on context:

| Scenario | Level |
|----------|-------|
| Normal player action | `info` |
| Suspicious or unusual action | `warn` |
| Script error or failed operation | `error` |
| Server crash / data loss risk | `fatal` |
| Verbose debug tracing | `debug` |

Use the shorthand exports when appropriate:
```lua
exports.fmsdk:Info("default", "Player connected", { playerSource = source })
exports.fmsdk:Warn("inventory", "Unusual item quantity detected", { playerSource = source, item = itemName, count = count })
exports.fmsdk:Error("default", "Database save failed", { playerSource = source, error = err })
```

### 9. Remove legacy logger convars

**ox_lib logger convars** — remove from server.cfg:
```
set ox:logger "fivemanage"
set fivemanage:key "YOUR_API_KEY"
```

**Discord webhook convars** — remove from server.cfg:
```
set inventory:webhook "https://discord.com/api/webhooks/..."
# any other resource-specific webhook convars
```

**Add fmsdk convar:**
```
set FIVEMANAGE_LOGS_API_KEY "your_api_key"
ensure fmsdk
```

---

## Workflow

When asked to migrate logging in a resource, follow these steps **in order**. Do not start writing code until step 4 is complete.

### Step 1 — Discover

Search all server-side files for logging call sites:
- `lib.logger(` — ox_lib logger
- `PerformHttpRequest(` — check if the URL contains `discord.com/api/webhooks`
- Local webhook helper functions: common names include `SendToDiscord`, `DiscordLog`, `sendLog`, `TriggerDiscord`, `webhook(`
- ConVars referencing Discord URLs: search for `webhook` in ConVar names

### Step 2 — Categorize

Group every call site by proposed dataset using the signal table in Rule 7. Do not assign a dataset individually per call — assign at the group level.

### Step 3 — Ask the user (required gate)

Present the proposed dataset mapping (see Rule 7, Step 2 for the format). **Wait for the user to confirm or correct it before proceeding.** This is not optional — the user decides the final dataset names.

If all logs clearly belong to one domain and the resource name makes the intent obvious, you may propose a single dataset and ask for a simple yes/no. But always ask.

### Step 4 — Migrate

Using the confirmed dataset mapping:

1. Rewrite each `lib.logger` call using Rules 1–5.
2. Rewrite each Discord webhook call using Rule 6.
3. Decompose any string-formatted messages into metadata (Rule 5).
4. Delete Discord webhook helper functions entirely.
5. Verify `fxmanifest.lua` — remove `ox_lib` dependency if logging was its only use.
6. Update `server.cfg` — remove legacy convars, add `FIVEMANAGE_LOGS_API_KEY` and `ensure fmsdk` (Rule 9).

Do not migrate client-side code — fmsdk logging is server-side only.

---

## fxmanifest Dependency Note

If the resource uses `ox_lib` only for logging, you can remove it as a dependency after migration. If other ox_lib features are used, keep the dependency.

```lua
-- Remove this line ONLY if ox_lib is no longer used at all:
dependency 'ox_lib'
```

---

## References

- `references/config-examples.md` — server.cfg and config.json examples
- `references/best-practices.md` — structured logging patterns and advice
