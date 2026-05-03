# Security & Server Authority

> Upstream: <https://create.roblox.com/docs/scripting/security>
> Source: `content/en-us/scripting/security/index.md`, `client-server-boundary.md`, `defensive-design.md`, `network-ownership.md`, `security-tactics.md`

## Purpose
Roblox clients run on user machines and can be modified. Treat every byte from a client as adversarial. The single most important principle: **the server is authoritative for anything that matters** (currency, progression, combat outcomes, leaderboard scores). Clients send intent; servers decide what happens.

## The trust boundary

| Side | Trust level | Visible to attackers |
|---|---|---|
| Server | Trusted | No |
| Client | Untrusted | Yes — including all code, assets, RemoteEvent names, and ReplicatedStorage contents |

Anything in `ReplicatedStorage` and below is shipped to the client. Exploit clients can:
- Read all your client and shared LuaScripts.
- See and call any `RemoteEvent`/`RemoteFunction` with arbitrary arguments.
- Modify their own character's CFrame within the bounds of network ownership.
- Inspect every Instance and attribute that replicates.

They **cannot** read `ServerStorage`, `ServerScriptService`, or properties the engine doesn't replicate.

## The validation checklist for every `OnServerEvent`

```lua
remote.OnServerEvent:Connect(function(player, action, payload)
    -- 1. Type-check every argument
    if typeof(action) ~= "string" then return end
    if typeof(payload) ~= "table" then return end

    -- 2. Range / enum-check
    if action ~= "buy" and action ~= "sell" then return end
    if typeof(payload.itemId) ~= "number" or payload.itemId < 1 then return end

    -- 3. Authorization — does this player have the right to do this?
    if not playerOwns(player, payload.itemId) then return end

    -- 4. Rate-limit per player
    if isThrottled(player, action) then return end

    -- 5. NOW execute the actual game logic, server-side
    handleAction(player, action, payload)
end)
```

Skipping any of these turns your remote into a free API for attackers.

For non-trivial payloads, hand-rolling step 1 (type-check) and step 2 (range-check) gets verbose fast. The standard library for this is **`osyrisrblx/t`** — a runtime type checker that lets you declare the expected shape once and validate in one call. See [community-libraries.md → t](community-libraries.md).

## Network ownership

For physics performance, the server typically gives **network ownership** of a player's character (and parts they're touching) to that player's client. The owner simulates the part locally and broadcasts its state. This means:

- A malicious client can teleport their character anywhere within the world bounds.
- A client can ignore physics constraints on parts they own.

Defensive techniques:
- **Server-side movement validation:** every Heartbeat, compare reported position to last position; if Δ exceeds max-speed × dt + tolerance, snap them back or kick.
- **Don't grant ownership of damage-dealing parts to the player**. Set `BasePart:SetNetworkOwner(nil)` to keep the server in charge of important physics.
- **Don't trust client-reported hit detection.** The client can claim it hit anything. Re-do the raycast on the server.

```lua
projectile:SetNetworkOwner(nil)  -- server simulates this part
```

## Common attack patterns and defences

| Attack | Defence |
|---|---|
| Spamming RemoteEvent calls | Per-player rate limit (token bucket) |
| Inflated currency in payload | Server holds canonical values; ignore client's number |
| Buying with negative price | Validate `price >= 0` and look up server-side |
| Teleport hack | Server velocity-cap movement, snap back |
| Auto-aim | Server-side line-of-sight check on hits |
| Fake admin commands | Check `player.UserId` against an allowlist; never trust a "isAdmin" flag from the client |

## Filtering player-entered text

Anything a player types or names that another player will see (chat, signs, custom names) **must** go through `TextService:FilterStringAsync` before display. Skipping this is a fast track to moderation actions against your experience.

```lua
local TextService = game:GetService("TextService")

local ok, result = pcall(function()
    return TextService:FilterStringAsync(rawText, fromPlayer.UserId)
end)
if not ok then return end
local cleaned = result:GetNonChatStringForBroadcastAsync()
```

## Pitfalls

- **"Just trust the client this once."** Once is enough. Validate every time.
- **Hiding logic in obfuscated client scripts.** Exploit tools deobfuscate trivially. If it must be secret, it must be on the server.
- **Storing prices client-side.** Look up server-side every purchase.
- **Validating only on first event.** Validate every event — exploiters fire arbitrary things.
- **Forgetting unreliable remotes are still adversarial.** UnreliableRemoteEvent payloads are just as fakeable as RemoteEvent.
- **Granting permanent network ownership of game-critical parts.** Always `:SetNetworkOwner(nil)` for projectiles, doors, hazards.

## See also
- [client-server-model.md](client-server-model.md) — what runs where
- [events-signals.md](events-signals.md) — RemoteEvent vs RemoteFunction
- [datastores.md](datastores.md) — server-only persistence
- [live-ops.md](live-ops.md) — admin-allowlist pattern in full (Permissions module, role gating)
- [combat.md](combat.md), [inventory.md](inventory.md), [shops.md](shops.md), [pets.md](pets.md), [placement.md](placement.md) — server-side validation applied throughout the gameplay systems
- [community-libraries.md](community-libraries.md) — `t` runtime type checker for validating remote payloads
