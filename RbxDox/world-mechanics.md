# World Mechanics (SpawnLocation, Conveyors, Moving Platforms, Jump Pads)

> Upstream: <https://create.roblox.com/docs/reference/engine/classes/SpawnLocation> · BasePart `AssemblyLinearVelocity` and `SurfaceVelocity` properties
> Source: distilled from BasePart property reference and standard Roblox map-mechanic patterns

## Purpose
A handful of world-mechanic Instances and patterns aren't built around the player-interaction primitives (`ProximityPrompt`, `ClickDetector`) — they're properties of geometry that affect any character that touches them. This card collects them: **`SpawnLocation`** (where players appear), **conveyor belts** (`SurfaceVelocity` on anchored parts), **moving platforms** (kinematic bodies), **jump pads** (Touched + velocity), and **trigger volumes** (invisible parts that detect entry without colliding).

## SpawnLocation

A `SpawnLocation` is a BasePart subclass that the engine spawns players onto. Multiple SpawnLocations can exist; the engine picks one based on team affinity and other rules.

```lua
-- Properties
spawn.Enabled                    -- bool; if false, no one spawns here
spawn.Neutral                    -- bool; if true, accepts players regardless of team
spawn.TeamColor                  -- BrickColor; matches a Team's TeamColor
spawn.AllowTeamChangeOnTouch     -- bool; player touching this spawn switches to the spawn's team
spawn.Duration                   -- seconds the ForceField on spawn lasts (default 10)
```

The engine creates a `Decal` with the team-color symbol on the top face automatically when a SpawnLocation has team affinity.

### Patterns

**Standard map spawn:** drop SpawnLocations under `workspace.Spawns` (Folder), set `Anchored = true`, `Neutral = true`, `Enabled = true`. Done.

**Team-locked spawn:**
```lua
spawn.Neutral = false
spawn.TeamColor = BrickColor.new("Bright red")    -- must match a Team in `Teams`
```

**Team-changer pad:**
```lua
spawn.Neutral = false
spawn.TeamColor = BrickColor.new("Bright blue")
spawn.AllowTeamChangeOnTouch = true               -- touching this spawn assigns Blue team
```

**Programmatic respawn point** (force a player to spawn somewhere specific next death):
```lua
player.RespawnLocation = spawnLocation
```
Engine respects `RespawnLocation` if set, regardless of team.

### Pitfalls

- **No SpawnLocations in the map** → players spawn at world origin (0, 0, 0). If your origin is empty space or inside a wall, players fall or get stuck.
- **SpawnLocation not Anchored** → physics moves it; players spawn into chaos.
- **TeamColor doesn't match any Team** → spawn never gets used.
- **Disabled SpawnLocations are still in the random pool** if `Enabled = true` is the default; toggle Enabled, don't just hide them.
- **`AllowTeamChangeOnTouch` fires on every Touched** (anchored-with-character contact). Players walking past a team-change spawn get switched. Place them where players have to deliberately stand on them.

## Conveyor belts — `SurfaceVelocity`

For "anchored part that pushes things on top of it" without using physics motors, set the part's surface velocity. The engine treats `SurfaceVelocity` as a property of the surface that any rigid body in contact with experiences as a friction-driven push.

```lua
-- Anchored conveyor belt part
conveyor.Anchored = true
conveyor.AssemblyLinearVelocity = Vector3.new(0, 0, -10)   -- preferred modern property
-- (legacy: BodyVelocity instance — deprecated)
```

Wait — for an anchored part, `AssemblyLinearVelocity` is read-only and zero. The right tool is the **`SurfaceVelocity` of `BasePart`** itself (set via `:ApplyImpulse` per frame, OR using `AssemblyLinearVelocity` on an *unanchored* but constraint-pinned part, OR via the legacy `BodyVelocity`/`LinearVelocity` constraint).

The cleanest modern approach for a conveyor: an **anchored part** with a high-friction material, plus a **CollectionService-tagged** layer that a script tracks and applies linear impulse to overlapping bodies each Heartbeat:

```lua
-- Server Script for all parts tagged "Conveyor"
local CollectionService = game:GetService("CollectionService")
local RunService = game:GetService("RunService")

local CONVEYOR_SPEED = 20

local op = OverlapParams.new()
op.FilterType = Enum.RaycastFilterType.Exclude
op.FilterDescendantsInstances = {}

RunService.Heartbeat:Connect(function()
    for _, conveyor in CollectionService:GetTagged("Conveyor") do
        local dir = conveyor:GetAttribute("Direction") or Vector3.new(0, 0, -1)
        local touching = workspace:GetPartBoundsInBox(
            conveyor.CFrame * CFrame.new(0, conveyor.Size.Y/2 + 1, 0),
            Vector3.new(conveyor.Size.X, 2, conveyor.Size.Z),
            op
        )
        for _, part in touching do
            if not part.Anchored then
                part.AssemblyLinearVelocity = Vector3.new(
                    dir.X * CONVEYOR_SPEED,
                    part.AssemblyLinearVelocity.Y,
                    dir.Z * CONVEYOR_SPEED
                )
            end
        end
    end
end)
```

Tag any part `"Conveyor"` and add a `Direction` attribute (Vector3); the system picks it up. Pairs naturally with [tags.md](tags.md) and [attributes.md](attributes.md).

### Alternative: physics-driven treadmill
A spinning anchored cylinder (one massive cylinder pretending to be a belt loop) with `AssemblyAngularVelocity` set works visually but is computationally heavier. The Heartbeat-impulse approach above is the standard.

## Moving platforms

Platforms that move on a path (rotating, oscillating, elevators).

### Anchored kinematic platform (the simple way)
Anchored parts don't simulate; you `:PivotTo` them along a path. Players standing on them **don't auto-follow** — the platform moves out from under them. To carry players along:

```lua
-- Server Script under the platform
local platform = script.Parent
local Players = game:GetService("Players")

local function getStandingPlayers(): {Player}
    local op = OverlapParams.new()
    op.FilterType = Enum.RaycastFilterType.Exclude
    local above = workspace:GetPartBoundsInBox(
        platform.CFrame * CFrame.new(0, platform.Size.Y/2 + 3, 0),
        Vector3.new(platform.Size.X, 6, platform.Size.Z),
        op
    )
    local players = {}
    for _, p in above do
        local model = p:FindFirstAncestorOfClass("Model")
        local hum = model and model:FindFirstChildOfClass("Humanoid")
        if hum then
            local player = Players:GetPlayerFromCharacter(model)
            if player then players[player] = model end
        end
    end
    return players
end

-- Move along a sine wave on Y
local startCF = platform.CFrame
local t = 0
game:GetService("RunService").Heartbeat:Connect(function(dt)
    t += dt
    local oldPivot = platform:GetPivot()
    platform:PivotTo(startCF * CFrame.new(0, math.sin(t) * 5, 0))

    -- Carry standing characters
    local delta = platform:GetPivot().Position - oldPivot.Position
    for _, char in getStandingPlayers() do
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if hrp then hrp.CFrame = hrp.CFrame + delta end
    end
end)
```

### Unanchored physics platform
Make the platform unanchored, attach via `AlignPosition` or `LinearVelocity` constraints to a target path. Players standing on it inherit motion naturally because the engine treats it as a regular rigid body in contact. Heavier on physics; simpler scripting.

For complex paths (rails, splines), the anchored+kinematic approach gives you exact frame-perfect control.

### Pitfalls
- **Anchored platform without character-carrying script** → players slide off as it moves.
- **Network ownership of unanchored platforms** with players on them — server should retain ownership for consistency, otherwise the riding player simulates it.
- **Sudden CFrame snaps** (teleport rather than continuous move) cause players to fall through. For elevators that "warp," briefly disable collisions or anchor the player.

## Jump pads

A part that launches whoever touches it.

```lua
-- Server Script under the pad
local pad = script.Parent
local LAUNCH = Vector3.new(0, 100, 0)
local debounce = {}

pad.Touched:Connect(function(hit)
    local model = hit:FindFirstAncestorOfClass("Model")
    local hum = model and model:FindFirstChildOfClass("Humanoid")
    if not hum or debounce[hum] then return end
    debounce[hum] = true

    local hrp = model:FindFirstChild("HumanoidRootPart")
    if hrp then
        hrp.AssemblyLinearVelocity = LAUNCH
    end

    task.delay(0.5, function() debounce[hum] = nil end)
end)
```

For directional launches, use `pad.CFrame.UpVector * power + pad.CFrame.LookVector * forward`. For "redirect any incoming velocity," compute reflection.

For a tagged jump-pad system, use the same CollectionService pattern as Conveyors.

## Trigger volumes (invisible detection)

A part configured to detect contact without affecting physics or rendering. Already covered in [collisions.md](collisions.md), repeated here for completeness:

```lua
local trigger = Instance.new("Part")
trigger.Anchored = true
trigger.CanCollide = false      -- doesn't block movement
trigger.CanQuery = false        -- doesn't show in raycasts
trigger.Transparency = 1        -- invisible
trigger.Size = Vector3.new(20, 10, 20)
trigger.Parent = workspace.Triggers

trigger.Touched:Connect(function(hit) ... end)
trigger.TouchEnded:Connect(function(hit) ... end)
```

For region-based detection without `Touched` (which requires at least one unanchored party), use per-frame `workspace:GetPartBoundsInBox` — see [collisions.md](collisions.md) and [raycasting.md](raycasting.md).

## Pickups (tag-based pattern)

Common map-mechanic combination: tag pickups, set up handler, destroy on touch:

```lua
-- Tag every pickup part as "Pickup"
-- One server script handles them all:
local CollectionService = game:GetService("CollectionService")

local function setupPickup(part: BasePart)
    local debounce = false
    part.Touched:Connect(function(hit)
        if debounce then return end
        local hum = hit.Parent and hit.Parent:FindFirstChildOfClass("Humanoid")
        local player = hum and game:GetService("Players"):GetPlayerFromCharacter(hum.Parent)
        if not player then return end
        debounce = true

        grantPickup(player, part:GetAttribute("ItemId"))
        part:Destroy()
    end)
end

for _, p in CollectionService:GetTagged("Pickup") do setupPickup(p) end
CollectionService:GetInstanceAddedSignal("Pickup"):Connect(setupPickup)
```

Designer-driven: tag a part `"Pickup"`, set its `ItemId` Attribute, done. See [tags.md](tags.md) for the broader pattern.

## Resource nodes (mining, gathering, harvesting)

A **resource node** is a re-usable world object that grants items when the player interacts with it, then enters a cooldown / respawn state. Iron ore that respawns 60 seconds after being mined; berry bush that regrows; tree that drops wood. Same shape across mining/farming/gathering subgenres.

```lua
-- Server Script that handles every part tagged "ResourceNode"
local CollectionService = game:GetService("CollectionService")
local TweenService = game:GetService("TweenService")

local function setupNode(node: BasePart)
    local prompt = node:FindFirstChildOfClass("ProximityPrompt") or createPrompt(node)
    prompt.ActionText = "Gather"
    prompt.HoldDuration = node:GetAttribute("GatherTime") or 1.5

    local available = true     -- node state

    prompt.Triggered:Connect(function(player)
        if not available then return end

        -- Tool tier check (e.g., copper pickaxe required for Iron node)
        local requiredTier = node:GetAttribute("RequiredToolTier") or 0
        if not playerHasToolTier(player, requiredTier) then
            notifyPlayer(player, "You need a higher-tier tool")
            return
        end

        -- Grant resource
        local itemId = node:GetAttribute("ItemId") or "Wood"
        local count = node:GetAttribute("YieldCount") or 1
        Inventory.Add(profile.inventory, itemId, count)

        -- Grant gathering XP if you have a separate progression sub-tree
        Progression.GrantXp(player, "Gathering", node:GetAttribute("Xp") or 1)

        -- Mark depleted, hide visually, schedule respawn
        available = false
        local respawnSec = node:GetAttribute("RespawnSec") or 60
        TweenService:Create(node, TweenInfo.new(0.3), {Transparency = 1}):Play()
        node.CanCollide = false
        prompt.Enabled = false

        task.delay(respawnSec, function()
            available = true
            TweenService:Create(node, TweenInfo.new(0.3), {Transparency = 0}):Play()
            node.CanCollide = true
            prompt.Enabled = true
        end)
    end)
end

for _, n in CollectionService:GetTagged("ResourceNode") do setupNode(n) end
CollectionService:GetInstanceAddedSignal("ResourceNode"):Connect(setupNode)
```

Designer workflow: place a part in the world, tag it `"ResourceNode"`, set Attributes:
- `ItemId` — what it gives ("IronOre", "Wood", "Berry")
- `YieldCount` — how many per gather (default 1)
- `GatherTime` — hold-duration in seconds (default 1.5)
- `RequiredToolTier` — minimum tool tier (default 0)
- `RespawnSec` — cooldown in seconds (default 60)
- `Xp` — gathering XP granted (default 1)

The script auto-wires every node from the tag. Add a node in Studio in 30 seconds, no code changes.

For visual variants (depleted-stump model swap, partially-mined ore mesh), parent multiple sub-models under the node and toggle Visibility / Transparency rather than using one part.

For **infinite-yield nodes with no respawn** (water source), skip the depletion step. For **destruction-on-deplete nodes** (one-shot crate), `:Destroy()` the node and remove its tag.

## Pitfalls (cross-cutting)

- **Touched-driven launchers / pads with no debounce** → fires every physics step in contact, launches the player into the stratosphere or grants 100x the intended item.
- **Per-frame velocity assignment on a player's HumanoidRootPart** fights the Humanoid's locomotion. For a sustained push, set `AssemblyLinearVelocity` once, not every Heartbeat.
- **Setting velocity on an Anchored part** silently does nothing.
- **Anchored moving platform with `CanCollide = true`** can crush players if it moves into them — the engine has no "elevator squash" protection. Use shorter movement steps or detect overlap and push the player out.
- **Server-side velocity on a player owned client-side** → the client overwrites it on the next replication tick. Either grant server ownership (`SetNetworkOwner(nil)`) for the duration, or fire a RemoteEvent and let the client do the launch.

## See also
- [inventory.md](inventory.md) — pickups and resource nodes grant into inventory
- [crafting.md](crafting.md) — gathered resources feed crafting recipes
- [collisions.md](collisions.md) — Touched, CanCollide/CanQuery/CanTouch, trigger patterns
- [spawn-mechanics.md](spawn-mechanics.md) — full spawn flow built on the SpawnLocation Instance covered here
- [areas-and-biomes.md](areas-and-biomes.md) — region/area system that combines trigger volumes with per-area state swaps
- [growth-systems.md](growth-systems.md) — plants/crops as a state-tagged-plot variant of the resource-node pattern
- [time-trials.md](time-trials.md) — start/progress/end gates as ordered trigger volumes for timed runs
- [environmental-effects.md](environmental-effects.md) — touch-triggered state-change blocks (kill, heal, buff, coin, speed, teleport, checkpoint)
- [tags.md](tags.md) — CollectionService for designer-driven mechanic tagging
- [attributes.md](attributes.md) — per-Instance config (Direction, ItemId, etc.)
- [raycasting.md](raycasting.md) — overlap queries for kinematic carrying
- [physics-constraints.md](physics-constraints.md) — alternative for moving via constraints (LinearVelocity, AlignPosition)
- [interactions.md](interactions.md) — for player-initiated interactions (ProximityPrompt) instead of passive touch
- [vehicles.md](vehicles.md) — VehicleSeat / Seat for sittable map elements
