# Passive vs Patched FiveM Logging

## Passive listener checklist

A separate logging resource can listen passively when all of these are true:

- The target action emits a server event with `RegisterNetEvent` or `TriggerEvent`.
- The event parameters include enough context to make the log useful.
- The event fires only after, or close enough to, the real action being logged.
- Duplicate logging is acceptable if multiple resources listen to the same event.

Passive listeners are best for framework events, police/job events, robbery lifecycle events, and server-local `TriggerEvent` calls.

## Patch checklist

Patch the target resource when any of these are true:

- The action happens inside `lib.addCommand`.
- The action happens inside `lib.callback.register` and no downstream framework event contains the missing context.
- The action is an export function with no event emitted.
- The action is client-only but needs an audit trail.
- Passive events exist but omit critical data such as admin actor, item name, exact amount, or result status.

## Where to place a patch

Place audit hooks after permission checks and after the action succeeds.

Good locations:

- After `RemoveMoney`, `AddMoney`, `AddItem`, `RemoveItem` returns success.
- After a DB insert/update callback succeeds.
- After a player is kicked, banned, jailed, revived, promoted, or fired.
- After `exports.qbx_core:AddPermission` / `RemovePermission` returns.

Avoid locations:

- Before permission checks, unless logging denied attempts.
- Before validation of client-supplied values.
- Inside tight loops or per-frame client threads.

## Event patch naming

Use a stable namespaced event name that belongs to the logging integration, not to upstream Qbox:

```lua
TriggerEvent('fm-logs:server:qbx_adminmenu:adminAction', source, payload)
TriggerEvent('fm-logs:server:qbx_vehicleshop:vehiclePurchased', source, payload)
```

This avoids changing the semantics of upstream events and makes the patch clearly removable.

## Direct fmsdk vs internal audit event

Prefer an internal audit event when maintaining an external logging resource like `fm-qbox`:

```lua
TriggerEvent('fm-logs:server:qbx_adminmenu:adminAction', source, payload)
```

Advantages:

- Upstream Qbox resource does not need a direct `fmsdk` dependency.
- Logging can be disabled or rerouted centrally.
- Patch is tiny and easy to review.

Use direct `exports.fmsdk:*` only when the target resource should own its logging and ship with Fivemanage support built in.
