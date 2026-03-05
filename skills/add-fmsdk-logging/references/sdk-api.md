# fmsdk Logging API Reference

Source: https://github.com/fivemanage/sdk/blob/main/README.md

---

## Installation

1. Download `fmsdk.zip` from the [release page](https://github.com/fivemanage/sdk/releases/latest)
2. Extract and place the `fmsdk` folder into your server's `resources` folder
3. Add to `server.cfg`:
   ```
   set FIVEMANAGE_LOGS_API_KEY your_api_key
   ensure fmsdk
   ```

> `screenshot-basic` is only required for image/video features — it is not needed for logging-only setups.

---

## Server Exports (Logging)

All logging exports are **server-side only**.

### `Log` — full control

```typescript
Log(
  datasetId: string,
  level: string,
  message: string,
  metadata?: {
    playerSource?: string | number,
    targetSource?: string | number,
    [key: string]: unknown
  }
): void
```

**Lua:**
```lua
exports.fmsdk:Log("default", "info", "Player connected", {
    playerSource = source,
    customData = "Additional info"
})

-- Without metadata
exports.fmsdk:Log("default", "error", "An error occurred")
```

**JavaScript:**
```javascript
exports.fmsdk.Log("default", "info", "Player connected", {
  playerSource: player.id,
  customData: "Additional info"
});

// Without metadata
exports.fmsdk.Log("default", "error", "An error occurred");
```

---

### `Info` — shorthand for level `"info"`

```typescript
Info(
  datasetId: string,
  message: string,
  metadata?: { playerSource?: string | number, targetSource?: string | number, [key: string]: unknown }
): void
```

```lua
exports.fmsdk:Info("inventory", "Item added to inventory", {
    playerSource = source,
    item = itemName,
    count = amount
})
```

---

### `Warn` — shorthand for level `"warn"`

```typescript
Warn(
  datasetId: string,
  message: string,
  metadata?: { playerSource?: string | number, targetSource?: string | number, [key: string]: unknown }
): void
```

```lua
exports.fmsdk:Warn("inventory", "Suspicious item quantity detected", {
    playerSource = source,
    item = itemName,
    amount = amount
})
```

---

### `Error` — shorthand for level `"error"`

```typescript
Error(
  datasetId: string,
  message: string,
  metadata?: { playerSource?: string | number, targetSource?: string | number, [key: string]: unknown }
): void
```

```lua
exports.fmsdk:Error("default", "Database save failed", {
    playerSource = source,
    error = err
})
```

---

## Datasets

The first argument of every log export is `datasetId` — the name of the dataset as configured in the Fivemanage dashboard.

```lua
-- Logs to the "bank" dataset
exports.fmsdk:Log("bank", "info", "Player transferred money", { ... })

-- Logs to the "default" dataset
exports.fmsdk:Log("default", "info", "Server started", { ... })
```

Datasets must be created in the Fivemanage dashboard before logs will appear there.

---

## Special Metadata Fields

| Field | Effect |
|-------|--------|
| `playerSource` | Triggers automatic appending of all player identifiers (license, steam, discord, IP, etc.) |
| `targetSource` | Same as `playerSource` but for a second player involved in the event |

These identifiers are appended automatically by the SDK when `appendPlayerIdentifiers` is `true` in `config.json` (default).

---

## Log Levels

Valid values (ordered most to least critical):

| Level | Description |
|-------|-------------|
| `fatal` | Critical failure threatening server stability or data |
| `error` | Failure affecting functionality (DB errors, failed saves) |
| `warn` | Unusual but non-breaking events |
| `info` | Normal operational events |
| `debug` | Verbose development tracing |

The minimum captured level is set by `"level"` in `config.json`. Logs below that level are dropped silently.
