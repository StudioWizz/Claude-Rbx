# Spawn Mechanics

> Source: distilled from spawn-flow patterns across PvP / round-based / RPG Roblox games

## Purpose
"Spawning" is more than `Players:LoadCharacter()`. A polished spawn flow involves **selecting** a spawn point (random from a pool, team-aware, away-from-enemies), **protecting** the new character for a few seconds (ForceField), **showing UI** before/around the spawn (death-cam → respawn-timer → spawn button → loadout select), and **applying loadout** (gear, abilities) at spawn time. This card pulls those together. For the engine-managed `SpawnLocation` Instance and the `Players.RespawnTime` setting, see [world-mechanics.md](world-mechanics.md). For the lifecycle events (`PlayerAdded`, `CharacterAdded`), see [player-lifecycle.md](player-lifecycle.md). For the Humanoid that spawns, see [characters.md](characters.md).

## The default flow vs the manual flow

**Default** (CharacterAutoLoads = true, default):
1. Player joins → engine spawns Character at a random `SpawnLocation` matching team/Neutral.
2. Player dies → engine waits `Players.RespawnTime` (default 5s) → respawns at a SpawnLocation.

**Manual** (`Players.CharacterAutoLoads = false`):
1. Player joins → no character yet.
2. Your code decides when and where to spawn → `player:LoadCharacter()`.
3. Player dies → no automatic respawn → your code triggers `:LoadCharacter()` again at the right moment.

Manual mode unlocks: pre-spawn UI, loadout selection, custom spawn-point algorithms, death-cam. Switch to it for any non-trivial game:

```lua
game:GetService("Players").CharacterAutoLoads = false
```

## Spawn-point selection

### Random from a pool (most games)
```lua
local Players = game:GetService("Players")
local CollectionService = game:GetService("CollectionService")

local function pickSpawnRandom(): SpawnLocation?
    local spawns = CollectionService:GetTagged("ActiveSpawn")
    if #spawns == 0 then return nil end
    return spawns[math.random(#spawns)]
end
```

Tag the spawn parts you want active (`"ActiveSpawn"`); designers add/remove the tag without code changes. See [tags.md](tags.md).

### Team-aware
```lua
local function pickSpawnForTeam(team: Team?): SpawnLocation?
    local pool = {}
    for _, s in CollectionService:GetTagged("ActiveSpawn") do
        if not team or s.Neutral or s.TeamColor == team.TeamColor then
            table.insert(pool, s)
        end
    end
    if #pool == 0 then return nil end
    return pool[math.random(#pool)]
end
```

`SpawnLocation.TeamColor` matching `Team.TeamColor` is the engine's team-spawn convention. See [world-mechanics.md](world-mechanics.md).

### Smart spawn (away from enemies)
For PvP, spawning into the middle of an enemy team is bad UX. Score each candidate by distance from the nearest enemy and pick the safest:

```lua
local function pickSpawnAwayFromEnemies(player: Player): SpawnLocation?
    local enemies = {}
    for _, p in Players:GetPlayers() do
        if p ~= player and p.Team ~= player.Team and p.Character then
            local hrp = p.Character:FindFirstChild("HumanoidRootPart")
            if hrp then table.insert(enemies, hrp.Position) end
        end
    end

    local best, bestScore = nil, -math.huge
    for _, s in CollectionService:GetTagged("ActiveSpawn") do
        if s.TeamColor ~= player.TeamColor and not s.Neutral then continue end

        local nearest = math.huge
        for _, ePos in enemies do
            nearest = math.min(nearest, (s.Position - ePos).Magnitude)
        end
        -- Prefer farther; clamp to avoid extreme bias
        local score = math.min(nearest, 200)
        if score > bestScore then best, bestScore = s, score end
    end
    return best
end
```

Tune the scoring (line-of-sight check, recent-death-location avoidance, "safe room" bonus) per game.

### Forced spawn point
Override for revival mechanics (resurrected at the place you died, or at a teammate):
```lua
player.RespawnLocation = specificSpawnLocation
-- Engine respects this on the next :LoadCharacter, regardless of team
```

## Spawn the character

Once you've picked the spawn point:

```lua
local function spawnPlayer(player: Player, atSpawn: SpawnLocation?)
    if atSpawn then player.RespawnLocation = atSpawn end
    player:LoadCharacter()
    -- After this, player.Character is a fresh model parented under Workspace.
    -- CharacterAdded fires; per-spawn setup (loadout, forcefield, etc.) goes there.
end
```

## Spawn protection — ForceField

A `ForceField` Instance parented under the Character renders a glow and applies `Humanoid:TakeDamage` immunity. The engine grants one automatically (`SpawnLocation.Duration = 10` seconds default); for non-default flows, manage it yourself:

```lua
player.CharacterAdded:Connect(function(character)
    local hum = character:WaitForChild("Humanoid")

    local ff = Instance.new("ForceField")
    ff.Visible = true
    ff.Parent = character

    -- Remove ForceField after delay OR on first attack action (whichever first)
    local attacked = false
    local function removeFF()
        if ff.Parent then ff:Destroy() end
    end

    -- Listen for the player taking an offensive action — they lose protection
    -- (Connect to your combat input remote, ability use, or shoot event.)
    local conn
    conn = AbilityUsedRemote.OnServerEvent:Connect(function(p)
        if p == player and not attacked then attacked = true; removeFF(); conn:Disconnect() end
    end)

    task.delay(SPAWN_PROTECTION_SEC, function()
        removeFF()
        if conn then conn:Disconnect() end
    end)
end)
```

The "removed on attack" rule prevents campers from sitting in invulnerable mode while shooting at vulnerable players.

## Pre-spawn UI flow (death → cam → respawn → loadout)

For polished games, between dying and re-spawning the player sees:
1. **Death cam** — camera locks on the killer or pans the scene for a few seconds
2. **Respawn timer** — countdown ("Respawn in 5… 4… 3…")
3. **Respawn button** — player presses Space to spawn (or auto-spawn at 0)
4. **Loadout / class select** — optional menu before spawn

```lua
-- Server
Players.CharacterAutoLoads = false

local DEATH_CAM_SEC = 2
local RESPAWN_DELAY = 5

Players.PlayerAdded:Connect(function(player)
    showLoadoutOnSpawn(player)
    player.CharacterAdded:Connect(function(char)
        local hum = char:WaitForChild("Humanoid")
        hum.Died:Connect(function()
            handleDeath(player, hum)
        end)
    end)
end)

local function handleDeath(player: Player, hum: Humanoid)
    local killerName = findKiller(hum) or "the world"
    local killerCharacter = findKillerCharacter(hum)

    -- Death cam (client-side)
    DeathCamRemote:FireClient(player, {killer = killerCharacter, durationSec = DEATH_CAM_SEC})

    task.wait(DEATH_CAM_SEC)
    -- Show respawn timer
    RespawnTimerRemote:FireClient(player, {seconds = RESPAWN_DELAY})

    task.wait(RESPAWN_DELAY)
    -- Optionally show loadout select; await selection
    ShowLoadoutRemote:FireClient(player)
    -- (LoadoutChosenRemote.OnServerEvent fires when player confirms; could also auto-spawn after timeout)
end

-- When loadout chosen
LoadoutChosenRemote.OnServerEvent:Connect(function(player, loadoutId)
    if not validateLoadout(player, loadoutId) then return end
    spawnWithLoadout(player, loadoutId)
end)

local function spawnWithLoadout(player: Player, loadoutId: string)
    local spawn = pickSpawnAwayFromEnemies(player)
    spawnPlayer(player, spawn)
    applyLoadout(player, loadoutId)
end
```

For client-side death-cam:
```lua
DeathCamRemote.OnClientEvent:Connect(function(data)
    local cam = workspace.CurrentCamera
    cam.CameraType = Enum.CameraType.Scriptable
    -- Look at the killer
    if data.killer then
        cam.CFrame = CFrame.lookAt(
            data.killer.HumanoidRootPart.Position + Vector3.new(0, 5, 10),
            data.killer.HumanoidRootPart.Position
        )
    end
    task.delay(data.durationSec, function()
        cam.CameraType = Enum.CameraType.Custom
    end)
end)
```

See [camera.md](camera.md) for the camera details, [menu-systems.md](menu-systems.md) for the loadout-select menu.

## Apply loadout at spawn

```lua
local function applyLoadout(player: Player, loadoutId: string)
    local loadout = LOADOUTS[loadoutId]
    if not loadout then return end

    local char = player.Character; if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid"); if not hum then return end

    -- Health / stats override
    hum.MaxHealth = loadout.maxHealth or 100
    hum.Health = hum.MaxHealth
    hum.WalkSpeed = loadout.walkSpeed or 16
    hum:SetAttribute("BaseWalkSpeed", hum.WalkSpeed)    -- for status-effects.md to read

    -- Grant tools
    player.Backpack:ClearAllChildren()
    for _, toolId in loadout.tools do
        local tool = ServerStorage.Tools[toolId]:Clone()
        tool.Parent = player.Backpack
    end

    -- Apply outfit (HumanoidDescription)
    if loadout.descriptionId then
        local desc = Players:GetHumanoidDescriptionFromOutfitId(loadout.descriptionId)
        hum:ApplyDescription(desc)
    end

    -- Reset abilities, status effects, etc.
    StatusEffects.Clear(hum)
    Combat.ResetState(player)
end
```

## Patterns

### "Press Space to spawn" gate
After death-cam and the respawn timer, don't auto-spawn — wait for the player to press a key. Reduces players spawning into bad situations because they were AFK during the timer:

```lua
-- Client
RespawnTimerRemote.OnClientEvent:Connect(function(data)
    showCountdown(data.seconds)
    task.wait(data.seconds)
    showSpawnPrompt("Press Space to respawn")

    UserInputService.InputBegan:Connect(function(input)
        if input.KeyCode == Enum.KeyCode.Space then
            RequestSpawnRemote:FireServer()
        end
    end)
end)
```

### Spawn at a teammate (squad games)
Allow the player to spawn near a living teammate instead of at base:
```lua
local function pickSpawnNearTeammate(player: Player): CFrame?
    for _, p in Players:GetPlayers() do
        if p.Team == player.Team and p ~= player and p.Character then
            local hrp = p.Character:FindFirstChild("HumanoidRootPart")
            if hrp then return hrp.CFrame * CFrame.new(math.random(-5, 5), 0, math.random(-5, 5)) end
        end
    end
    return nil
end

-- Override RespawnLocation isn't enough (it's a SpawnLocation). For arbitrary CFrame:
player:LoadCharacter()
task.wait()    -- wait one frame for character to exist
local hrp = player.Character.HumanoidRootPart
hrp.CFrame = pickSpawnNearTeammate(player) or hrp.CFrame
```

Validate teammate isn't in active combat; offer this as a UI button after respawn timer.

### Last-known-position revival (single-player / co-op)
Save the player's CFrame periodically; on respawn, spawn them at the saved CFrame instead of a SpawnLocation. Useful for big open-world games where SpawnLocation is far from where you died:

```lua
local lastSafe: {[Player]: CFrame} = {}

RunService.Heartbeat:Connect(function()
    for _, p in Players:GetPlayers() do
        if p.Character and isInSafeArea(p.Character) then
            local hrp = p.Character:FindFirstChild("HumanoidRootPart")
            if hrp then lastSafe[p] = hrp.CFrame end
        end
    end
end)

-- On respawn:
local cf = lastSafe[player]
if cf then
    player:LoadCharacter()
    task.wait()
    player.Character.HumanoidRootPart.CFrame = cf
end
```

### Spawn camera intro
For dramatic spawns: camera flies in from above, lands behind the character. Use TweenService on `cam.CFrame` over 1.5–2s. See [tweens-animation.md](tweens-animation.md), [camera.md](camera.md).

### Reserved / private match spawn
After a TeleportService teleport into a reserved server, spawn into the appropriate map zone based on the `TeleportData`. See [teleport.md](teleport.md).

### Stagger spawns at round start
For round-based games where everyone spawns at the same moment, briefly disable `Touched` events and physics interactions for a half-second so players don't damage-collide instantly:
```lua
for _, p in Players:GetPlayers() do
    spawnPlayer(p)
    local char = p.Character
    for _, part in char:GetDescendants() do
        if part:IsA("BasePart") then part.CanCollide = false end
    end
    task.delay(0.5, function()
        for _, part in char:GetDescendants() do
            if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then part.CanCollide = true end
        end
    end)
end
```

See [round-based.md](round-based.md).

## Pitfalls

- **`Players.CharacterAutoLoads = false` set without ever calling `:LoadCharacter`.** Players join, never spawn, sit at the world origin invisibly. Test the manual flow before shipping it.
- **Spawning while a previous Character still exists.** Calling `:LoadCharacter` while the old character lives causes the engine to destroy the old one — fine for respawn, weird if you intended to keep it. Check `player.Character` first.
- **Spawn-point pool empty.** No SpawnLocations tagged → `:LoadCharacter` falls back to world origin (often a wall or the void). Always have a fallback `Vector3.new(0, 50, 0)` if the pool is empty.
- **Smart-spawn algorithm runs every spawn for everyone.** O(spawns × enemies) per spawn — fine for small games, slow with many players. Cache enemy positions per Heartbeat or sample-only.
- **ForceField removed on the wrong action.** "Walk forward" shouldn't drop protection; "shoot" should. Wire to specific offensive remotes, not generic input.
- **ForceField never removed.** Player walks around invulnerable forever. Always `:Destroy()` it (with the timeout fallback).
- **Death-cam camera not restored.** Camera stuck Scriptable forever. Restore `CameraType = Custom` after the cam ends.
- **Loadout grants Tools to a Backpack that's about to be cleared.** Clear `Backpack` *first*, then grant. Otherwise the new tools vanish.
- **Spawn near teammate without distance/safety check.** Player spawns inside an enemy. Validate the teammate is alive AND not in combat AND not in a tight space.
- **Respawn timer accelerated by client claim.** Trust server timing; the client UI displays the server's countdown.
- **Spawn protection that allows the player to interact with objectives.** Capture-the-flag style: the spawn-protected player grabs the flag, becomes invulnerable until they "attack." Drop protection on objective-interaction too.
- **Respawning during round transitions.** Player dies at round-end → spawns into intermission map → looks broken. Suppress respawn during non-Active states. See [round-based.md](round-based.md).
- **`player:LoadCharacter()` called rapidly.** Each call destroys+recreates the character, which is heavy and produces visual flicker. Debounce.
- **`RespawnLocation` set without checking it still exists.** A removed SpawnLocation gives `:LoadCharacter` nothing to spawn at → world origin. Re-validate or use the random pool as fallback.

## See also
- [areas-and-biomes.md](areas-and-biomes.md) — per-area spawn pools; respawn at last-safe-area
- [characters.md](characters.md) — Humanoid, `:LoadCharacter`, the Character anatomy that spawns
- [player-lifecycle.md](player-lifecycle.md) — `PlayerAdded`, `CharacterAdded`, `Died` — the events spawn flows hook into
- [world-mechanics.md](world-mechanics.md) — `SpawnLocation` Instance: TeamColor, Neutral, AllowTeamChangeOnTouch
- [resource-bars.md](resource-bars.md) — health/stamina/mana bars rendered for the spawned character
- [equipment.md](equipment.md), [tools.md](tools.md), [inventory.md](inventory.md) — loadout application
- [avatar-customization.md](avatar-customization.md) — outfit applied at spawn
- [camera.md](camera.md) — death-cam camera control
- [menu-systems.md](menu-systems.md) — loadout-select menu, respawn UI
- [round-based.md](round-based.md) — staggered spawns at round start; suppress respawn outside Active
- [teleport.md](teleport.md) — spawn into reserved-server matches via TeleportData
- [security.md](security.md) — server-side spawn-point validation; client can't pick arbitrary positions
