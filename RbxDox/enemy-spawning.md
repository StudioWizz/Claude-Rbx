# Enemy Spawning

> Source: distilled from PvE / horde / RPG / tower-defense enemy spawn patterns

## Purpose
`npcs.md` covers what an enemy IS (Model + Humanoid + AI script + network ownership). This card covers **how enemies appear in the world**: spawner Instances tagged in the map, wave-based budgets, per-spawner cooldowns, max-concurrent caps to keep performance predictable, weighted random pools, per-area filtering ([areas-and-biomes.md](areas-and-biomes.md)), and despawn-when-far rules. Without these, naive spawning either floods the server (every spawner ticks every Heartbeat) or fails to populate the world at all.

## Spawner Instance shape

A spawner is a tagged BasePart (often invisible) marking where enemies appear. Designer drops them in the map; one server script handles all of them.

```lua
-- In Studio: place an invisible Part at each spawn location
spawner.Anchored = true
spawner.CanCollide = false
spawner.CanQuery = false
spawner.Transparency = 1
spawner.Size = Vector3.new(2, 2, 2)
spawner:AddTag("EnemySpawner")

-- Designer-tunable Attributes:
spawner:SetAttribute("EnemyPool", "Wolf,Boar,Bear")           -- comma-separated; weighted via PoolWeights
spawner:SetAttribute("PoolWeights", "70,25,5")                -- matches EnemyPool order
spawner:SetAttribute("MaxConcurrent", 3)                       -- max alive enemies from THIS spawner
spawner:SetAttribute("CooldownSec", 15)                        -- minimum seconds between spawns
spawner:SetAttribute("SpawnRadius", 5)                         -- random offset from spawner position
spawner:SetAttribute("ActivationRadius", 100)                  -- only spawn if a player is within this
spawner:SetAttribute("DespawnRadius", 200)                     -- despawn if all players are beyond this
spawner:SetAttribute("AreaId", "ForestGlade")                  -- optional: tie to areas-and-biomes.md
spawner:SetAttribute("Difficulty", "Normal")                   -- optional: tie to enemy-difficulty.md
```

The pattern across this card: tags + attributes drive everything. Designers author spawners in Studio with no code changes.

## Spawner manager — server-side core

```lua
--!strict
local CollectionService = game:GetService("CollectionService")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")

type SpawnerState = {
    spawner: BasePart,
    activeEnemies: {[Model]: true},        -- alive enemies this spawner has spawned
    lastSpawnAt: number,                    -- os.clock()
}

local states: {[BasePart]: SpawnerState} = {}

local function setupSpawner(spawner: BasePart)
    states[spawner] = {
        spawner = spawner,
        activeEnemies = {},
        lastSpawnAt = 0,
    }
end

for _, s in CollectionService:GetTagged("EnemySpawner") do setupSpawner(s) end
CollectionService:GetInstanceAddedSignal("EnemySpawner"):Connect(setupSpawner)
CollectionService:GetInstanceRemovedSignal("EnemySpawner"):Connect(function(s) states[s] = nil end)
```

## Pool selection (weighted random)

```lua
local function parseList(s: string): {string}
    local result = {}
    for item in s:gmatch("[^,]+") do table.insert(result, item:match("^%s*(.-)%s*$")) end
    return result
end

local function pickFromPool(spawner: BasePart): string?
    local poolStr = spawner:GetAttribute("EnemyPool"); if not poolStr or poolStr == "" then return nil end
    local weightsStr = spawner:GetAttribute("PoolWeights") or ""
    local pool = parseList(poolStr)
    local weights = parseList(weightsStr)

    if #pool == 0 then return nil end

    local total = 0
    local nums: {number} = {}
    for i = 1, #pool do
        local w = tonumber(weights[i]) or 1
        nums[i] = w
        total += w
    end

    local roll = math.random() * total
    local accum = 0
    for i = 1, #pool do
        accum += nums[i]
        if roll <= accum then return pool[i] end
    end
    return pool[#pool]
end
```

## Distance check (activation / despawn)

```lua
local function nearestPlayerDistance(pos: Vector3): number
    local nearest = math.huge
    for _, p in Players:GetPlayers() do
        local hrp = p.Character and p.Character:FindFirstChild("HumanoidRootPart")
        if hrp then nearest = math.min(nearest, (hrp.Position - pos).Magnitude) end
    end
    return nearest
end
```

## The spawn tick

Throttled to ~2Hz — spawners don't need every-frame precision. Each tick, for each spawner: check activation radius, check max-concurrent, check cooldown, then maybe spawn.

```lua
local TICK_INTERVAL = 0.5
local lastTick = 0

local function trySpawn(state: SpawnerState)
    local spawner = state.spawner
    local now = os.clock()

    -- Cooldown
    local cooldown = spawner:GetAttribute("CooldownSec") or 10
    if now - state.lastSpawnAt < cooldown then return end

    -- Max concurrent (count alive)
    local maxConcurrent = spawner:GetAttribute("MaxConcurrent") or 3
    local count = 0
    for enemy in state.activeEnemies do
        if not enemy.Parent or not enemy:FindFirstChildOfClass("Humanoid") then
            state.activeEnemies[enemy] = nil
        else
            count += 1
        end
    end
    if count >= maxConcurrent then return end

    -- Activation radius
    local activation = spawner:GetAttribute("ActivationRadius") or 100
    if nearestPlayerDistance(spawner.Position) > activation then return end

    -- Pick from pool
    local enemyId = pickFromPool(spawner); if not enemyId then return end

    -- Spawn
    local radius = spawner:GetAttribute("SpawnRadius") or 0
    local offset = Vector3.new((math.random() * 2 - 1) * radius, 0, (math.random() * 2 - 1) * radius)
    local enemy = spawnEnemy(enemyId, spawner.Position + offset, {
        difficulty = spawner:GetAttribute("Difficulty"),
        areaId = spawner:GetAttribute("AreaId"),
    })
    if enemy then
        state.activeEnemies[enemy] = true
        state.lastSpawnAt = now
        -- Track despawn back-reference so the death/despawn handler clears the slot
        enemy:SetAttribute("SourceSpawnerPath", spawner:GetFullName())
    end
end

local function tickDespawn(state: SpawnerState)
    -- Despawn enemies that strayed too far OR if all players left the area
    local despawnRadius = state.spawner:GetAttribute("DespawnRadius") or 200
    for enemy in state.activeEnemies do
        if not enemy.Parent then continue end
        local hrp = enemy:FindFirstChild("HumanoidRootPart") :: BasePart?
        if not hrp then continue end
        if nearestPlayerDistance(hrp.Position) > despawnRadius then
            enemy:Destroy()
            state.activeEnemies[enemy] = nil
        end
    end
end

RunService.Heartbeat:Connect(function()
    local now = os.clock()
    if now - lastTick < TICK_INTERVAL then return end
    lastTick = now

    for _, state in states do
        trySpawn(state)
        tickDespawn(state)
    end
end)
```

The two ticks (spawn + despawn) run together. Spawn budgets in, dead enemies out. The world maintains a steady population of enemies near players, and is empty far away — exactly what you want for performance.

## Hooking despawn back to the spawner

When an enemy dies, its slot in the spawner should free up. Two ways:

**A. Via Humanoid.Died event** (cleaner):
```lua
local function spawnEnemy(enemyId: string, position: Vector3, opts): Model?
    local enemy = NPC_TEMPLATES[enemyId]:Clone()
    enemy:PivotTo(CFrame.new(position))
    enemy.Parent = workspace.Enemies

    -- Apply difficulty (see enemy-difficulty.md)
    if opts.difficulty then applyDifficulty(enemy, opts.difficulty) end

    -- Hook death
    local hum = enemy:FindFirstChildOfClass("Humanoid")
    if hum then
        hum.Died:Connect(function()
            task.wait(3)        -- corpse linger
            local sourcePath = enemy:GetAttribute("SourceSpawnerPath")
            local source = sourcePath and resolveByFullName(sourcePath)
            if source and states[source] then
                states[source].activeEnemies[enemy] = nil
            end
            enemy:Destroy()
        end)
    end

    return enemy
end
```

**B. Via the per-tick sweep** (already shown in `trySpawn`): walking `state.activeEnemies` and clearing entries with destroyed/missing Humanoids. Works without the explicit Died hook but is reactive (only frees slots on the next tick).

Use both for safety — the Died hook is fast; the sweep is the catch-all.

## Wave-based spawning (horde mode, tower defense)

Round-based games spawn in deliberate waves rather than continuously. The pattern: a `WaveSpawner` Instance defines a queue, each wave specifies enemy types + counts + delay between spawns + delay between waves.

```lua
type Wave = {
    enemies: {[string]: number},        -- enemyId → count
    intervalSec: number,                 -- seconds between spawns within the wave
    delayAfterWave: number?,             -- seconds after this wave before next starts
}

local WAVES = {
    {enemies = {Wolf = 5}, intervalSec = 1.5, delayAfterWave = 10},
    {enemies = {Wolf = 8, Boar = 2}, intervalSec = 1.2, delayAfterWave = 15},
    {enemies = {Bear = 3}, intervalSec = 3, delayAfterWave = 20},
    {enemies = {Wolf = 15, Bear = 2, AlphaWolf = 1}, intervalSec = 0.8},
}

local function runWave(spawner: BasePart, wave: Wave)
    -- Round-robin through enemy types weighted by remaining count
    local remaining = {}
    for id, count in wave.enemies do remaining[id] = count end

    while next(remaining) do
        local pool = {}
        for id, count in remaining do for _ = 1, count do table.insert(pool, id) end end
        if #pool == 0 then break end
        local enemyId = pool[math.random(#pool)]
        remaining[enemyId] -= 1
        if remaining[enemyId] <= 0 then remaining[enemyId] = nil end

        spawnEnemy(enemyId, spawner.Position, {})
        task.wait(wave.intervalSec)
    end
end

-- Wave manager
task.spawn(function()
    for waveNum, wave in WAVES do
        broadcastWaveStart(waveNum)
        runWave(spawner, wave)
        if wave.delayAfterWave then
            broadcastWaveEnd(waveNum, wave.delayAfterWave)
            task.wait(wave.delayAfterWave)
        end
    end
end)
```

Pair with [round-based.md](round-based.md): the wave manager is the round's "Active" state's behaviour; `checkWin` returns the round winner when waves complete or players die.

For tower defense specifically, enemies follow a path (use [characters.md](characters.md) `PathfindingService` from spawn point to a goal); when an enemy reaches the goal, deduct lives.

## Per-area enemy filtering

Combine with [areas-and-biomes.md](areas-and-biomes.md): a spawner's `AreaId` attribute restricts what spawns. Or, spawners outside any area inherit the player's current area filter:

```lua
-- In trySpawn, intersect spawner's pool with the area's allowed enemies
local function filterByArea(pool: {string}, areaId: string?): {string}
    if not areaId then return pool end
    local areaPool = AREA_PRESETS[areaId] and AREA_PRESETS[areaId].enemyPool
    if not areaPool then return pool end

    local allowed = {}
    for _, e in areaPool do allowed[e] = true end

    local result = {}
    for _, e in pool do if allowed[e] then table.insert(result, e) end end
    return result
end
```

So a spawner tagged with both `EnemyPool = "Wolf,Bear,Dragon"` and `AreaId = "ForestGlade"` only spawns enemies that are *both* in its pool AND allowed in the Forest area. Designer authors broadly; area config narrows.

## Patterns

### Group / pack spawning
A "Wolf Pack" spawn = 1 alpha + 3-5 wolves at once, not a stream of solo wolves. Override `trySpawn` to spawn a configured group atomically:

```lua
spawner:SetAttribute("SpawnAsGroup", true)
spawner:SetAttribute("GroupSize", 4)
-- In trySpawn: instead of one enemy, spawn `GroupSize` from pool, scattered around the spawner
```

### Boss spawner
A `BossSpawner` tag with `MaxConcurrent = 1` and `CooldownSec = 600`. Boss death triggers a timer; only respawns after a long cooldown. Often paired with a server-wide announcement via [live-ops.md](live-ops.md).

### Trigger-on-enter spawner (ambush)
Spawner ignores cooldown and continuous spawning; instead, a player entering its zone fires it once:

```lua
spawnerTriggerVolume.Touched:Connect(function(hit)
    if hit.Parent:FindFirstChildOfClass("Humanoid") and not state.triggered then
        state.triggered = true
        for _ = 1, ambushCount do spawnEnemy(...) end
    end
end)
```

Reset `state.triggered = false` after a cooldown OR on boss death.

### Pooled spawning (object pool)
For high-spawn-rate situations (waves of weak enemies), don't `:Destroy` and re-create — pool. Reuse a small set of enemy Models, hide them when "dead," reposition and re-enable for next spawn. See [performance.md](performance.md) object-pool pattern.

### Time-of-day gating
Some enemies (zombies, vampires) only spawn at night. Check `time-and-weather.md` `GameTime.GetHour()` in `trySpawn` and skip spawning during day:
```lua
if enemyId == "Zombie" and GameTime.GetHour() > 6 and GameTime.GetHour() < 20 then return end
```

### Spawn budget across all spawners
In addition to per-spawner `MaxConcurrent`, enforce a *total* enemy cap across the server (e.g., max 50 enemies alive). Add a check at the top of `trySpawn`:
```lua
local totalAlive = 0
for _, state in states do for _ in state.activeEnemies do totalAlive += 1 end end
if totalAlive >= MAX_ALIVE_TOTAL then return end
```

Protects against runaway scenarios where 30 spawners each cap at 5 = 150 enemies and the server lags.

### Per-spawner difficulty override
Tagged `EliteSpawner` enemies always spawn at +1 difficulty tier. `Difficulty` attribute on the spawner overrides per-area defaults. See [enemy-difficulty.md](enemy-difficulty.md).

### Server-shared spawner state (cross-server)
For "global boss is dead in every server" coordination, persist boss respawn time in MemoryStore + announce via MessagingService. See [live-ops.md](live-ops.md) and [memorystores.md](memorystores.md).

## Pitfalls

- **Per-Heartbeat tick across hundreds of spawners.** Throttle to 2Hz; even slower (0.2Hz) for non-combat ambient spawners (peaceful wildlife).
- **No max-concurrent cap.** A spawner with no activation-radius check or no cap floods the area. Always cap.
- **Activation radius too small.** Players walk into an empty area — no enemies appear because they spawn just-in-time. Set activation > combat-engagement range so enemies are *already there* when the player arrives.
- **Despawn radius too small.** Enemies vanish in the player's peripheral vision. Set despawn > activation by a comfortable margin (e.g., activation 100, despawn 200).
- **Despawning during combat.** Enemy despawned mid-fight because the player ran past the despawn radius for one frame. Add a "combat lock" — don't despawn enemies that have been damaged by a player in the last N seconds.
- **`Humanoid.Died` not freeing the spawner slot.** Check via the per-tick sweep too; don't rely on Died alone (NPCs can be removed by other systems without dying).
- **Pool string parsing breaking on whitespace.** "Wolf, Boar , Bear" trips up naive `:split(",")`. Use the trim pattern shown in `parseList`.
- **Weighted pool with all-zero weights.** Total = 0 → division by zero. Guard.
- **Spawning inside geometry.** Enemy appears stuck in a wall. Either raycast down from `spawner.Position + Vector3.new(0, 50, 0)` to find ground, or make sure spawners are placed where there's clear space.
- **Spawning multiple enemies at the exact same position.** They tangle in physics. Use `SpawnRadius` jitter, OR queue spawns with a small delay between.
- **Network ownership of spawned enemies.** Default goes to nearest player → they can manipulate the enemy's physics. Always `:SetNetworkOwner(nil)` for hostile enemies. See [security.md](security.md).
- **Wave manager that doesn't survive script restart.** A `task.spawn` while-loop dies if the script is disabled. Persist current wave to MemoryStore if you need recovery.
- **Spawner `Touched`-driven (ambush) firing on every body part.** Multiple character parts touch the trigger → multiple ambushes. Dedupe by tracking which player has triggered.
- **Spawned enemy script doesn't know its spawner.** Can't update spawner state on death. Set `SourceSpawnerPath` attribute at spawn and look it up.
- **Forgetting `applyDifficulty` per spawn.** Enemies always spawn at base stats regardless of `Difficulty` attribute. Apply on spawn.

## See also
- [npcs.md](npcs.md) — what an enemy IS (composition, AI, network ownership)
- [enemy-difficulty.md](enemy-difficulty.md) — applying difficulty stat scaling on spawn
- [collisions.md](collisions.md) — selective barriers (enemies pass through walls players can't); Selective Barriers section
- [areas-and-biomes.md](areas-and-biomes.md) — per-area enemy pools that filter spawner choices
- [round-based.md](round-based.md) — wave-based gameplay using this card's spawn primitives
- [tags.md](tags.md), [attributes.md](attributes.md) — designer-driven spawner setup
- [performance.md](performance.md) — pooling enemies for high-spawn-rate gameplay
- [security.md](security.md) — `:SetNetworkOwner(nil)` for hostile NPCs
- [characters.md](characters.md) — pathfinding from spawn point to goal (tower-defense paths)
- [time-and-weather.md](time-and-weather.md) — time-of-day spawn gating
- [live-ops.md](live-ops.md) — cross-server boss respawn coordination
- [memorystores.md](memorystores.md) — persisted boss timers across servers
- [combat.md](combat.md) — what the spawned enemies actually do once alive
