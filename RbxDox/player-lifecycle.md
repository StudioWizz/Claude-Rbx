# Player Lifecycle & AFK

> Upstream: <https://create.roblox.com/docs/reference/engine/classes/Players> · <https://create.roblox.com/docs/reference/engine/classes/Player>
> Source: distilled from the Player / Players / Humanoid / DataModel API references

## Purpose
Roblox surfaces a small set of events that mark every important transition in a player's session: joining the server, spawning a character, dying, going AFK, leaving, and the server itself shutting down. Wiring these up correctly is what makes a game feel polished — saves persist, characters get their loadout, AFK players don't hold matchmaking slots, and shutdowns don't lose data. This card is the consolidated lifecycle reference.

## The full timeline

```
                                   ┌─ Humanoid.Died
                                   ├─ Humanoid.HealthChanged
                                   ├─ Humanoid.Idled (after ~2 minutes still on character)
                                   ├─ Humanoid.Seated, FreeFalling, Jumping, etc.
                                   │
PlayerAdded ─→ CharacterAdded ─→ ─┴─→ CharacterRemoving ─→ CharacterAdded ─→ ... ─→ PlayerRemoving
                                                          (respawn cycle)
   │
   ├─ Player.Idled (after ~20 minutes of NO INPUT)
   └─ player.AncestryChanged → leaves
```

And separately, on shutdown:
```
game:BindToClose(callback)   -- runs once when the server is stopping; callback has ~30s
```

## Core events

### `Players.PlayerAdded` (server)
Fires when a player connects. Use for: load data, set up per-player state, send welcome RemoteEvent.

```lua
local Players = game:GetService("Players")

Players.PlayerAdded:Connect(function(player)
    print(player.Name, "joined")
    loadProfile(player)               -- DataStore
    player.RespawnLocation = chooseSpawn(player)
end)
```

**Gotcha:** if your script starts AFTER players have joined (e.g., a new script added in a long-running session, or hot-reloaded), `PlayerAdded` won't retroactively fire. Guard with:
```lua
for _, player in Players:GetPlayers() do
    onPlayerAdded(player)
end
Players.PlayerAdded:Connect(onPlayerAdded)
```

### `Players.PlayerRemoving` (server)
Fires when a player disconnects (any reason — quit, kicked, crashed, teleported away). Use for: save data, clean up server-side state.

```lua
Players.PlayerRemoving:Connect(function(player)
    saveProfile(player)
end)
```

`PlayerRemoving` runs **before** the player is actually removed from `Players:GetPlayers()`. Yields here delay the player's actual departure briefly.

### `player.CharacterAdded` (server or client)
Fires when a Character spawns — first spawn after join AND every respawn after death.

```lua
player.CharacterAdded:Connect(function(character)
    local hum = character:WaitForChild("Humanoid")
    hum.Health = MAX_HEALTH
    giveStartingTools(player)
    applyAvatarOverride(hum)
end)

-- Cover the first character if it already exists when this script runs
if player.Character then onCharacterAdded(player.Character) end
```

### `player.CharacterRemoving` (server or client)
Fires when a Character is about to be destroyed (death, respawn, manual `:LoadCharacter`). Use for cleanup of per-character state.

### `humanoid.Died`
Fires when Health reaches 0.

```lua
humanoid.Died:Connect(function()
    -- broadcast death, drop loot, stat tracking
end)
```

After Died, the engine waits `Players.RespawnTime` (default 5 s) then auto-respawns (unless `Players.CharacterAutoLoads = false`).

### `Players.CharacterAutoLoads`
Set `false` if you want to control respawn manually:
```lua
Players.CharacterAutoLoads = false

humanoid.Died:Connect(function()
    task.wait(3)
    player:LoadCharacter()      -- manually trigger respawn
end)
```

### `humanoid.Idled` (≠ `Player.Idled`)
Fires after ~2 minutes of the character not moving. Useful for "afk emote when standing still" — NOT a true AFK detector (player may still be clicking and looking around).

```lua
humanoid.Idled:Connect(function(time)
    -- time = seconds idle
    -- play idle emote, etc.
end)
```

## AFK detection — `Player.Idled`

The reliable AFK signal. Fires when Roblox detects ~20 minutes of **no input** (mouse, keyboard, gamepad, touch — any).

```lua
Players.PlayerAdded:Connect(function(player)
    player.Idled:Connect(function(idleTime)
        -- idleTime: seconds since last input
        markAfk(player, idleTime)
    end)
end)
```

`Player.Idled` re-fires every ~30 seconds while the player remains AFK, with cumulative idle time. Use the latest value.

### Custom AFK with shorter threshold

The built-in 20-minute threshold is too long for many use cases (matchmaking, leaderboards, party games). Roll your own using `UserInputService` on the client and replicate to server:

```lua
-- LocalScript in StarterPlayerScripts
local UIS = game:GetService("UserInputService")
local Players = game:GetService("Players")
local AFK_REMOTE = game.ReplicatedStorage:WaitForChild("AfkStatus")

local AFK_THRESHOLD = 60        -- seconds
local lastInput = os.clock()
local isAfk = false

local function reportAfk(state: boolean)
    if state == isAfk then return end
    isAfk = state
    AFK_REMOTE:FireServer(state)
end

UIS.InputBegan:Connect(function() lastInput = os.clock(); reportAfk(false) end)
UIS.InputChanged:Connect(function() lastInput = os.clock(); reportAfk(false) end)

game:GetService("RunService").Heartbeat:Connect(function()
    if os.clock() - lastInput > AFK_THRESHOLD then reportAfk(true) end
end)
```

```lua
-- Server
local afkPlayers: {[Player]: boolean} = {}
AFK_REMOTE.OnServerEvent:Connect(function(player, isAfk)
    if typeof(isAfk) ~= "boolean" then return end
    afkPlayers[player] = isAfk or nil
    -- update icon, drop from matchmaking queue, etc.
end)
```

**Caveat:** the client reports AFK status — they could lie (claim active when AFK to keep matchmaking slot). For competitive uses, also track server-observed input proxies (movement, last RemoteEvent fired, etc.) and trust the maximum.

### AFK kick

```lua
Players.PlayerAdded:Connect(function(player)
    player.Idled:Connect(function(idleTime)
        if idleTime > 30 * 60 then           -- 30 minutes
            player:Kick("Disconnected due to inactivity.")
        end
    end)
end)
```

## Reconnect handling

Players who disconnect mid-session may rejoin. Without coordination they get a fresh DataStore read while another server may still be saving — race conditions. The discipline:

- **Use session locking** — store a session ID in the player's DataStore on load; reject loads if another session ID is active and recent. This is what `ProfileService` does for you (see [community-libraries.md](community-libraries.md)).
- **Save promptly on PlayerRemoving + BindToClose** — minimize the window where stale data could be read.
- **On rejoin, the new server must wait** for the old server's save to commit. Session locking gives you a deterministic way to detect "the previous session hasn't released yet" and back off / retry.

See [datastores.md](datastores.md) for the full save/load pattern.

## `BindToClose` — graceful shutdown

```lua
game:BindToClose(function()
    -- This runs when the server is shutting down (manual stop, scaling, place update)
    -- You have ~30 seconds before forced termination
    local pending = {}
    for _, player in Players:GetPlayers() do
        table.insert(pending, task.spawn(function()
            saveProfile(player)
        end))
    end
    -- Wait for all to finish (with timeout safety)
    for _, t in pending do
        if coroutine.status(t) ~= "dead" then task.wait() end
    end
end)
```

`BindToClose` only runs in **published places**. In Studio play-test it doesn't fire reliably — test in production for confidence.

If your save is slow and exceeds ~30 s, you'll lose data. Save in parallel (each player in a `task.spawn`), and design saves to be small enough to fit.

## Common patterns

### Welcome flow (data-load → grant items → spawn)
```lua
local function onPlayerAdded(player)
    local profile = loadProfileSync(player)        -- yields
    if not profile then
        player:Kick("Could not load your data. Try again later.")
        return
    end

    player.CharacterAdded:Connect(function(char)
        local hum = char:WaitForChild("Humanoid")
        hum.Health = profile.maxHealth
        applyOutfit(hum, profile.outfit)
        for _, itemId in profile.inventory do giveTool(player, itemId) end
    end)

    -- Force first spawn now that data is loaded (if CharacterAutoLoads = false)
    player:LoadCharacter()
end
```

### Death streak tracking
```lua
local deaths: {[Player]: number} = {}

Players.PlayerAdded:Connect(function(player)
    deaths[player] = 0
    player.CharacterAdded:Connect(function(char)
        local hum = char:WaitForChild("Humanoid")
        hum.Died:Connect(function()
            deaths[player] += 1
            if deaths[player] >= 3 then
                showHelpUI(player)
            end
        end)
    end)
end)

Players.PlayerRemoving:Connect(function(player) deaths[player] = nil end)
```

### Drop AFK players from a matchmaking queue
```lua
Players.PlayerAdded:Connect(function(player)
    player.Idled:Connect(function()
        removeFromMatchmaking(player)
    end)
end)
```

### Periodic autosave
```lua
task.spawn(function()
    while true do
        task.wait(120)         -- every 2 minutes
        for _, p in Players:GetPlayers() do
            task.spawn(function() saveProfile(p) end)
        end
    end
end)
```
Autosave reduces data loss to at most one cycle's worth, even if `BindToClose` is interrupted.

## Pitfalls

- **`PlayerAdded` doesn't fire for already-joined players** if your script starts late. Always loop `:GetPlayers()` first.
- **`CharacterAdded` may not have replicated children yet on the client.** Use `:WaitForChild("Humanoid")`.
- **`player.Character` is `nil` between death and respawn.** Always check before indexing.
- **Yielding inside `PlayerRemoving`** delays the player's actual departure — useful for save-to-completion, dangerous if you yield indefinitely.
- **`BindToClose` doesn't run in Studio reliably.** Test save logic in published places.
- **`Player.Idled` AFK time accumulates across multiple firings.** Don't expect a one-shot.
- **Confusing `Humanoid.Idled` with `Player.Idled`** — Humanoid.Idled means "character not moving for 2 minutes," Player.Idled means "no input for 20 minutes." Different signals, different uses.
- **Forgetting to clean up per-player tables on `PlayerRemoving`** → memory leaks across long-running servers.
- **Saving in `Heartbeat`** → DataStore quota exhaustion within minutes. Save on lifecycle events and a long-period autosave.
- **Trusting client-reported AFK status** without server cross-check → players game it for matchmaking advantage.
- **Setting `CharacterAutoLoads = false` then forgetting to call `LoadCharacter`** → players spawn invisibly with no Character. The default true is the right value unless you have a specific reason.

## See also
- [datastores.md](datastores.md) — save patterns and BindToClose discipline
- [save-slots.md](save-slots.md) — multi-save extension; lifecycle hooks need slot-aware saves
- [offline-processing.md](offline-processing.md) — accrue rewards on PlayerAdded based on saved `lastOnline`
- [characters.md](characters.md) — Humanoid events (Died, HealthChanged, Idled)
- [spawn-mechanics.md](spawn-mechanics.md) — full spawn/respawn flow that lifecycle events drive
- [project-structure.md](project-structure.md) — where lifecycle scripts live (ServerScriptService)
- [security.md](security.md) — server-side validation of any client-reported state (including AFK)
- [memorystores.md](memorystores.md) — cross-server presence (e.g., "is this user online anywhere?")
- [community-libraries.md](community-libraries.md) — **ProfileService** does session locking and reconnect-safe saves for you
