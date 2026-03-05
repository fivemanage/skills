# fmsdk Configuration Examples

---

## server.cfg

```
# Fivemanage API key for logging
set FIVEMANAGE_LOGS_API_KEY "your_logs_api_key_here"

ensure fmsdk
```

> Get your API key from the Fivemanage dashboard under **Tokens** — create a token with type `Logs`.

> `screenshot-basic` is only required for image/video features. For logging-only setups, you do not need it.

---

## config.json (inside the fmsdk resource)

Full example with all logging-relevant options:

```json
{
  "logs": {
    "level": "info",
    "levels": ["debug", "info", "warn", "error", "fatal"],
    "console": false,
    "enableCloudLogging": true,
    "appendPlayerIdentifiers": true,
    "excludedPlayerIdentifiers": ["ip", "live"],
    "playerEvents": true,
    "chatEvents": false,
    "txAdminEvents": true
  }
}
```

### Option reference

| Key | Default | Description |
|-----|---------|-------------|
| `level` | `"info"` | Minimum level to capture. Logs below this level are dropped. Must be a value in `levels`. |
| `levels` | `[...]` | Ordered hierarchy from most to least critical. Only these values are valid at runtime. |
| `console` | `false` | Print logs to server console. Enable during development. |
| `enableCloudLogging` | `true` | Send logs to Fivemanage cloud. Set `false` for local-only testing. |
| `appendPlayerIdentifiers` | `true` | Auto-append player identifiers when `playerSource` or `targetSource` is in metadata. |
| `excludedPlayerIdentifiers` | `[]` | Identifier types to exclude from auto-appending (e.g. `"ip"`, `"live"`, `"xbl"`). |
| `playerEvents` | `true` | Auto-log player connecting and dropped events. |
| `chatEvents` | `false` | Auto-log chat messages. Can generate very high log volume — keep disabled unless needed. |
| `txAdminEvents` | `true` | Auto-log txAdmin events (kicks, bans, restarts). |

---

## Development vs Production Config

### Development (verbose, local only)
```json
{
  "logs": {
    "level": "debug",
    "console": true,
    "enableCloudLogging": false,
    "playerEvents": true,
    "chatEvents": true,
    "txAdminEvents": true
  }
}
```

### Production (info+, cloud enabled, no chat noise)
```json
{
  "logs": {
    "level": "info",
    "console": false,
    "enableCloudLogging": true,
    "appendPlayerIdentifiers": true,
    "excludedPlayerIdentifiers": ["ip"],
    "playerEvents": true,
    "chatEvents": false,
    "txAdminEvents": true
  }
}
```

---

## fxmanifest.lua Dependency

Add `fmsdk` as a dependency in the resource being instrumented:

```lua
dependencies {
    'fmsdk'
}
```

---

## Dataset Setup

Datasets are created in the Fivemanage dashboard. Common setup:

| Dataset Name | Used for |
|-------------|----------|
| `default` | General server events, catch-all |
| `inventory` | Item add/remove, shops, stashes |
| `bank` | Money transfers, transactions |
| `admin` | Admin actions, bans, kicks |
| `vehicles` | Spawns, modifications, impounds |
| `combat` | Kills, damage, weapon events |

Reference datasets by name in log calls:
```lua
exports.fmsdk:Log("inventory", "info", "Item removed", { playerSource = source, item = "water", amount = 1 })
exports.fmsdk:Log("bank", "warn", "Large transfer detected", { playerSource = source, amount = 100000 })
```
