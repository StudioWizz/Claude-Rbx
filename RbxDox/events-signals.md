# Events & Signals

> Upstream: <https://create.roblox.com/docs/scripting/events>
> Source: `content/en-us/scripting/events/index.md`, `bindable.md`, `remote.md`, `deferred.md`

## Purpose
Almost everything reactive in Roblox is an event. There are four flavours, all with the same API shape but very different reach.

| Type | Crosses VM boundary? | Use for |
|---|---|---|
| `RBXScriptSignal` (built-in) | No | Engine events: `Touched`, `PlayerAdded`, `Heartbeat`, etc. |
| `BindableEvent` | No | Custom events within a single VM (server-to-server or client-to-client) |
| `BindableFunction` | No | Same, but with a return value |
| `RemoteEvent` | **Yes** (server ↔ client) | Network-replicated fire-and-forget messages |
| `RemoteFunction` | **Yes** (server ↔ client) | Network-replicated request/response (use sparingly) |
| `UnreliableRemoteEvent` | Yes, lossy | High-frequency state where dropped packets are fine (positions, etc.) |

## API shape (the same for all)
```lua
local conn = signal:Connect(function(...) end)  -- returns RBXScriptConnection
conn:Disconnect()

local ... = signal:Wait()                        -- yields, returns the next fire's args
signal:Once(function(...) end)                   -- auto-disconnects after first fire
```

For Bindable/Remote you also have:
```lua
bindable:Fire(...)
remote:FireServer(...)        -- client → server
remote:FireClient(player, ...) -- server → one client
remote:FireAllClients(...)    -- server → all
```

## RemoteEvent — the workhorse
```lua
-- ReplicatedStorage/Remotes/Damage  (RemoteEvent)
local DamageRemote = game.ReplicatedStorage.Remotes.Damage

-- Server
DamageRemote.OnServerEvent:Connect(function(player, targetId: number, amount: number)
    -- Validate: client can lie about anything!
    if typeof(targetId) ~= "number" or typeof(amount) ~= "number" then return end
    if amount < 0 or amount > 100 then return end
    -- Apply server-authoritative damage
end)

-- Client
DamageRemote:FireServer(targetId, 25)
```

## BindableEvent — decoupling within one side
Use to let a producer fire events without callers having to know about the consumer (like an event bus).
```lua
-- ServerStorage/Bus.lua
local Bus = Instance.new("BindableEvent")
return Bus

-- Producer
local Bus = require(game.ServerStorage.Bus)
Bus:Fire("RoundEnded", winningTeam)

-- Consumer
Bus.Event:Connect(function(name, ...)
    if name == "RoundEnded" then ... end
end)
```

## RemoteFunction — request/response (be careful)
```lua
-- Server
ShopRemote.OnServerInvoke = function(player, itemId)
    return canAfford(player, itemId), price(itemId)
end

-- Client
local ok, price = ShopRemote:InvokeServer("sword")
```
The client-side invoke yields until the server returns. If you `:InvokeClient` (server → client), a malicious client can yield forever and hang your server thread. **Don't InvokeClient unless you absolutely need it; prefer FireClient + a return RemoteEvent.**

## Deferred event behaviour
`Workspace.SignalBehavior` controls how event handlers schedule:
- `Deferred` (modern default) — handlers run on the next resumption point in batch; multiple fires within the same step coalesce in order without re-entrant surprises.
- `Immediate` (legacy) — handlers run synchronously inside `Fire`; can recurse and is harder to reason about.

Stick with `Deferred`. Write handlers that don't assume "this runs the instant I call Fire."

## UnreliableRemoteEvent

Same API as RemoteEvent but with no delivery guarantees: packets may be lost or arrive out of order. The trade is **lower latency and higher throughput** for "best effort" data.

### Why it exists (the protocol layer)

Roblox's network transport itself uses UDP under the hood, but the regular `RemoteEvent` adds a **reliability wrapper** on top: every packet is acknowledged, retried on loss, and delivered in order. That wrapper is what makes `RemoteEvent` reliable — and also what adds latency. `UnreliableRemoteEvent` skips the wrapper entirely, so packets go out the door immediately and never wait on retransmits.

### Rule of thumb

| Use a `RemoteEvent` when… | Use an `UnreliableRemoteEvent` when… |
|---|---|
| Missing the message changes game state (purchase, damage, score, "round started") | Missing one frame's worth is invisible to the player (continuous position, aim direction, particle steering) |
| Order matters (ability sequence, queued commands) | Order doesn't matter (each packet supersedes the last) |
| You fire occasionally (events, transitions) | You fire frequently (every Heartbeat, every input change) |

Phrase it as: **packet loss vs. packet delay — which hurts more?** If delay hurts more, go unreliable.

### Common pattern: split fire-extinguisher into two channels

```lua
-- Reliable: "I started extinguishing" — must arrive
ExtinguishStart:FireServer()

-- Unreliable: every Heartbeat while held, send aim direction
RunService.Heartbeat:Connect(function()
    if isExtinguishing then
        ExtinguishAim:FireServer(mouse.Hit.Position)   -- losing one frame is fine
    end
end)

-- Reliable: "I stopped" — must arrive
ExtinguishStop:FireServer()
```

The server uses `Start`/`Stop` to bound the effect's lifetime (those events must not be lost) and uses the rapid `Aim` packets to steer the particle direction (occasional drops are imperceptible).

Other good fits: free-aim camera direction sent to other clients, footstep particle positions, NPC blend-tree pose updates, third-person crosshair overlays.

### What to NOT use it for
- Damage application
- Currency / inventory changes
- Round state transitions
- Anything the server should react to exactly once

### Security note
Unreliable does NOT mean "less validated." Payloads are just as forgeable as a regular RemoteEvent — clamp, type-check, and rate-limit on the server identically. See [security.md](security.md).

Upstream announcement and discussion: <https://devforum.roblox.com/t/introducing-unreliableremoteevents/2724155>.

## Patterns

### Single-fire wait
```lua
local player = Players.PlayerAdded:Wait()
```

### Disconnect on cleanup
```lua
local conns = {}
table.insert(conns, part.Touched:Connect(...))
-- later
for _, c in conns do c:Disconnect() end
```

### Type the payload
Define a shared module with the schema:
```lua
-- ReplicatedStorage/Schemas.lua
export type DamageMsg = {targetId: number, amount: number}
return nil
```
Then validate against the type in handlers (Luau won't enforce it across the wire — you must check at runtime).

## Pitfalls

- **Connection leaks.** A connection holds the closure (and its upvalues) alive forever. Disconnect when the consumer is destroyed. `Instance.Destroying` and `:GetPropertyChangedSignal("Parent")` help.
- **Trusting `OnServerEvent` arguments.** Always validate types, ranges, and that the player has the right to do what they're asking.
- **Firing remotes every frame.** Each fire is a network packet. Throttle (e.g. 10–20 Hz) or use UnreliableRemoteEvent for high-frequency state.
- **`Wait()` in a hot path.** Yields the thread. For multiple events use `:Connect` and let the engine schedule.

## See also
- [client-server-model.md](client-server-model.md) — replication boundary
- [security.md](security.md) — validating remote payloads
- [task-scheduling.md](task-scheduling.md) — deferred vs immediate
- [community-libraries.md](community-libraries.md) — **Warp** for batched/high-throughput remotes
- [tools.md](tools.md), [interactions.md](interactions.md), [chat.md](chat.md) — common producers of player→server events
