# fmsdk Configuration Examples

---

## server.cfg

```
# Start screenshot-basic before fmsdk (required for image capture, harmless for logging-only setups)
ensure screenshot-basic
ensure fmsdk

# Fivemanage API key for logging
set FIVEMANAGE_LOGS_API_KEY "your_logs_api_key_here"

# If also using media (images/video) features:
# set FIVEMANAGE_MEDIA_API_KEY "your_media_api_key_here"
```

> Get your API key from the Fivemanage dashboard under **Tokens** — create a token with type `Logs`.

---

## config.json (fmsdk resource)

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
| `level` | `"info"` | Minimum level to capture. Logs below this level are dropped. Must be in `levels`. |
| `levels` | `[...]` | Ordered hierarchy from most to least critical. Only these values are valid at runtime. |
| `console` | `false` | Print logs to server console. Enable during development. |
| `enableCloudLogging` | `true` | Send logs to Fivemanage cloud. Set `false` for local-only testing. |
| `appendPlayerIdentifiers` | `true` | Auto-append player identifiers when `playerSource` or `targetSource` is in metadata. |
| `excludedPlayerIdentifiers` | `[]` | Identifier types to exclude from auto-appending (e.g. `"ip"`, `"live"`, `"xbl"`). |
| `playerEvents` | `true` | Auto-log player connecting and dropped events. |
| `chatEvents` | `false` | Auto-log chat messages. Can generate high log volume — use carefully. |
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

## ox_lib → fmsdk convar migration

**Remove from server.cfg:**
```
set ox:logger "fivemanage"
set fivemanage:key "YOUR_API_KEY"
```

**Add to server.cfg:**
```
set FIVEMANAGE_LOGS_API_KEY "YOUR_API_KEY"
ensure fmsdk
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

Reference datasets by name in your log calls:
```lua
exports.fmsdk:Log("inventory", "info", "Item removed", { playerSource = source, item = "water", amount = 1 })
exports.fmsdk:Log("bank", "warn", "Large transfer detected", { playerSource = source, amount = 100000 })
```
