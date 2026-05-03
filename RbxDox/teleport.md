# TeleportService

> Upstream: <https://create.roblox.com/docs/projects/teleport> · <https://create.roblox.com/docs/reference/engine/classes/TeleportService>
> Source: distilled from the TeleportService API and the matchmaking-handoff completion of the example in `memorystores.md`

## Purpose
`TeleportService` moves players between **places** (different `.rbxl` files within the same experience or in another experience), into **specific server instances** (matchmaking handoff), and into **reserved private servers**. It's the natural completion of the matchmaking-queue pattern — once your matchmaker has paired N players, you teleport them into a reserved server together.

## Key API

```lua
local TeleportService = game:GetService("TeleportService")

-- Modern, options-based (preferred)
TeleportService:TeleportAsync(placeId: number, players: {Player}, options: TeleportOptions?)

-- Reserved private servers
local accessCode, privateServerId = TeleportService:ReserveServer(placeId)
TeleportService:TeleportToPrivateServer(placeId, accessCode, players, spawnName?, teleportData?)

-- Legacy single-player (still works, less flexible)
TeleportService:Teleport(placeId, player)
TeleportService:TeleportToPlaceInstance(placeId, jobId, player)

-- Look up a friend's server
TeleportService:GetPlayerPlaceInstanceAsync(userId)   -- → success, errorMessage, placeId, jobId

-- Per-player persistent settings (survives across teleports within the experience)
TeleportService:SetTeleportSetting(name, value)
TeleportService:GetTeleportSetting(name)

-- Events
TeleportService.TeleportInitFailed:Connect(function(player, teleportResult, errMsg, placeId, options) end)
-- (in the SOURCE server)

-- On the TARGET server, in PlayerAdded handler:
local joinData = player:GetJoinData()
-- joinData.TeleportData : the table you passed
-- joinData.SourcePlaceId, joinData.SourceJobId, joinData.ReferredByPlayerId
```

## TeleportOptions

For `TeleportAsync` you build a `TeleportOptions` Instance to control the teleport:

```lua
local options = Instance.new("TeleportOptions")
options.ShouldReserveServer = true                           -- create a fresh reserved server for these players
options.ReservedServerAccessCode = accessCode                -- OR use an existing reserved server
options.ServerInstanceId = jobId                             -- OR target a specific known server instance
options:SetTeleportData({                                    -- payload received via player:GetJoinData() at the target
    matchId = "abc123",
    teamAssignments = {[42] = "Red", [99] = "Blue"},
})
TeleportService:TeleportAsync(placeId, players, options)
```

`TeleportOptions` is one-shot — build a new one per teleport.

## Patterns

### Lobby → matchmade game (completing the `memorystores.md` queue example)
```lua
local MSS = game:GetService("MemoryStoreService")
local TeleportService = game:GetService("TeleportService")

local q = MSS:GetQueue("Match", 30)
local MATCH_PLACE_ID = 1234567890

-- Matchmaker tick
task.spawn(function()
    while true do
        task.wait(5)
        local items, id = q:ReadAsync(8, false, 0)
        if not items or #items < 8 then continue end

        -- Resolve players (they must still be on this server, or use cross-server messaging)
        local players = resolveLocalPlayers(items)
        if #players < 8 then continue end

        local ok, err = pcall(function()
            local accessCode, _privateServerId = TeleportService:ReserveServer(MATCH_PLACE_ID)
            local options = Instance.new("TeleportOptions")
            options.ReservedServerAccessCode = accessCode
            options:SetTeleportData({
                matchId = game:GetService("HttpService"):GenerateGUID(false),
                roster = items,
            })
            TeleportService:TeleportAsync(MATCH_PLACE_ID, players, options)
        end)

        if ok then
            q:RemoveAsync(id)            -- ack only on successful teleport
        else
            warn("Teleport failed:", err)
            -- Don't ack: items reappear after invisibility timeout, retry next tick
        end
    end
end)
```

In the **target match server**:
```lua
local Players = game:GetService("Players")

Players.PlayerAdded:Connect(function(player)
    local data = player:GetJoinData()
    local teleportData = data.TeleportData
    if not teleportData then
        -- Player joined directly without going through the matchmaker — handle (kick to lobby?)
        return
    end
    -- Validate the payload (don't trust it blindly even though it came via Roblox)
    if typeof(teleportData.matchId) ~= "string" then return end
    enrollPlayerInMatch(player, teleportData.matchId)
end)
```

### Hub world with multiple minigame places
A common multi-place layout: a hub place where players congregate, and several minigame places. From the hub, a button or ProximityPrompt triggers a teleport.

```lua
local MINIGAMES = {
    Soccer  = 5555555555,
    Racing  = 5555555556,
    Parkour = 5555555557,
}

button.Activated:Connect(function()   -- LocalScript
    teleportRemote:FireServer("Soccer")
end)

teleportRemote.OnServerEvent:Connect(function(player, name)   -- Server
    local placeId = MINIGAMES[name]
    if not placeId then return end
    local options = Instance.new("TeleportOptions")
    options:SetTeleportData({fromHub = true})
    TeleportService:TeleportAsync(placeId, {player}, options)
end)
```

### Reserved private server (party / friend group)
```lua
local accessCode, privateServerId = TeleportService:ReserveServer(placeId)
-- Save accessCode somewhere (DataStore against the party leader's UserId), share with friends.
-- Friends call TeleportToPrivateServer with the accessCode.
TeleportService:TeleportToPrivateServer(placeId, accessCode, friendsList)
```

The same `accessCode` can be used to teleport additional players to the same reserved server within its lifetime.

### Handle teleport failures
```lua
TeleportService.TeleportInitFailed:Connect(function(player, result, err, placeId, options)
    -- result: Enum.TeleportResult — Failure, FlooderRateLimit, IsTeleporting, GameNotFound, etc.
    -- Show UI, log, optionally retry with backoff
    if result == Enum.TeleportResult.Flooded or result == Enum.TeleportResult.IsTeleporting then
        task.delay(2, function()
            TeleportService:TeleportAsync(placeId, {player}, options)
        end)
    end
end)
```

`TeleportInitFailed` fires in the **source** server. The target server hasn't been reached yet at this point.

### Loading screen during teleport
Show a `ScreenGui` to the player **before** calling teleport — Roblox's default loading screen takes over once the teleport begins. To customize the loading screen on arrival, use `ReplicatedFirst` in the target place (scripts under `ReplicatedFirst` run before the default UI).

## Pitfalls

- **`ReserveServer` is server-only and yields.** Wrap in `pcall`.
- **Studio play-test can't actually teleport.** TeleportService calls no-op or error in Studio. Test in a published place.
- **TeleportData is NOT secret.** It can be inspected by anyone in the target server's process (it's basically a JSON blob delivered by Roblox). Don't put secrets in it; treat it like a remote payload — validate at the target.
- **TeleportData maxes out around ~30 KB** of JSON. For bigger payloads, store on MemoryStore/DataStore and pass only an id.
- **Cross-experience teleport (different game)** — TeleportData is **dropped** when teleporting between unrelated experiences. Same-experience cross-place teleports preserve it.
- **`TeleportAsync` doesn't wait for arrival.** It returns once Roblox has accepted the request. Use `Players.PlayerRemoving` (source) and `PlayerAdded` (target) for the actual transitions.
- **Reserved server access codes don't expire immediately** but they're not infinite — typically usable for several hours. Don't store one in DataStore for a week and expect it to work.
- **Teleporting a player who's already teleporting** errors with `IsTeleporting`. Track per-player teleport state if you fire from multiple sources.
- **Players who can't be teleported** (offline, kicked) cause partial failures — `TeleportAsync` of multiple players may succeed for some and fail for others. Handle via `TeleportInitFailed` per player.
- **Don't teleport from `PlayerAdded`** without a yield — the player isn't fully loaded yet and the call may fail. Wait for `player.CharacterAdded:Wait()` or a small `task.wait`.
- **Quota:** experience-wide rate limits exist (~tens of teleports per minute per player). Bursts during matchmaking are fine; per-frame teleport calls will throttle.

## See also
- [areas-and-biomes.md](areas-and-biomes.md) — same-place areas as the alternative to splitting into separate places
- [memorystores.md](memorystores.md) — the matchmaking queue this teleport completes
- [messaging-http.md](messaging-http.md) — coordinate which server handles the teleport
- [security.md](security.md) — validate `TeleportData` at the target like any untrusted payload
- [project-structure.md](project-structure.md) — `ReplicatedFirst` for custom loading screens
- [services.md](services.md) — TeleportService overview
