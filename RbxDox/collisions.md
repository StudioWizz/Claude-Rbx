# Collisions

> Upstream: <https://create.roblox.com/docs/workspace/collisions>
> Source: `content/en-us/workspace/collisions.md`, `content/en-us/physics/network-ownership.md`

## Purpose
Collisions are how parts interact physically (push, block, bounce) and how scripts get notified (`Touched`). Roblox gives you four levers: per-part flags, collision groups, NoCollisionConstraint, and CanCollide tuning at runtime.

## The four CanX flags

| Flag | What it controls | Default |
|---|---|---|
| `CanCollide` | Physical collision (blocking) | `true` |
| `CanTouch` | `Touched` / `TouchEnded` events fire | `true` |
| `CanQuery` | Visible to raycasts and overlap queries | `true` |
| `CollisionGroup` | Named group; pairs of groups can be (non)collidable | `"Default"` |

Set `CanCollide = false` for triggers, decorative parts, or visual-only effects. Set `CanQuery = false` to make a part invisible to spatial queries (e.g., a sword's hitbox you never want to raycast against).

## Touched events

```lua
local conn = part.Touched:Connect(function(hit: BasePart)
    -- hit is the OTHER part that touched
end)

part.TouchEnded:Connect(function(hit) end)
```

`Touched` fires every physics step a contact exists, not once per touch. Always debounce — see [task-scheduling.md](task-scheduling.md).

`Touched` requires `CanTouch = true` and at least one of the parts to be unanchored (or both anchored with one of them setting `Touched`-relevant flags). For purely-anchored detection, use overlap queries instead.

## CollisionGroups

Define named groups in `PhysicsService` (server) so you can globally control which classes of parts collide with which:

```lua
local PhysicsService = game:GetService("PhysicsService")

PhysicsService:RegisterCollisionGroup("Players")
PhysicsService:RegisterCollisionGroup("Projectiles")

-- Players don't collide with their own projectiles
PhysicsService:CollisionGroupSetCollidable("Players", "Projectiles", false)

-- Assign parts
for _, p in characterModel:GetDescendants() do
    if p:IsA("BasePart") then p.CollisionGroup = "Players" end
end
projectile.CollisionGroup = "Projectiles"
```

Group setup must happen on the **server**; clients see the resulting collision matrix automatically.

`CollisionGroupContainsPart(group, part)` and `:GetCollisionGroups()` are available for inspection.

## NoCollisionConstraint

For one-off "these two specific parts don't collide" without a global group, use `NoCollisionConstraint` (see [physics-constraints.md](physics-constraints.md)). Faster than juggling groups for ad-hoc pairs.

## Selective barriers — one-way / faction-only walls

A common applied pattern: a barrier that **lets some entities through but blocks others**. Examples:
- A "spawn corridor" enemies use to enter the arena, but players can't enter
- A "team door" only Red Team can pass through
- A "phantom wall" Phantom-tagged units phase through; everything else blocks
- A "drop-down floor" players can fall through but enemies can't (asymmetric)

The mechanism: **CollisionGroups** (already covered above) plus a tag-driven assignment script so designers don't manually configure every part.

### Pattern: tagged barrier blocks specific groups

```lua
-- One-time setup at server start
local PhysicsService = game:GetService("PhysicsService")

PhysicsService:RegisterCollisionGroup("Players")
PhysicsService:RegisterCollisionGroup("Enemies")
PhysicsService:RegisterCollisionGroup("PlayerOnlyBarrier")     -- blocks players, lets enemies through
PhysicsService:RegisterCollisionGroup("EnemyOnlyBarrier")      -- blocks enemies, lets players through

-- Configure: barriers block one side
PhysicsService:CollisionGroupSetCollidable("Players", "PlayerOnlyBarrier", true)
PhysicsService:CollisionGroupSetCollidable("Enemies", "PlayerOnlyBarrier", false)
PhysicsService:CollisionGroupSetCollidable("Enemies", "EnemyOnlyBarrier", true)
PhysicsService:CollisionGroupSetCollidable("Players", "EnemyOnlyBarrier", false)

-- Players are in "Players" group, enemies are in "Enemies"
-- Each character/enemy script assigns the group on spawn:
local function assignPlayerCollisionGroup(character: Model)
    for _, p in character:GetDescendants() do
        if p:IsA("BasePart") then p.CollisionGroup = "Players" end
    end
end
local function assignEnemyCollisionGroup(model: Model)
    for _, p in model:GetDescendants() do
        if p:IsA("BasePart") then p.CollisionGroup = "Enemies" end
    end
end
```

Then designers tag barrier parts in Studio and a small script assigns the right collision group:

```lua
local CollectionService = game:GetService("CollectionService")

CollectionService:GetInstanceAddedSignal("PlayerOnlyBarrier"):Connect(function(p)
    if p:IsA("BasePart") then p.CollisionGroup = "PlayerOnlyBarrier" end
end)
CollectionService:GetInstanceAddedSignal("EnemyOnlyBarrier"):Connect(function(p)
    if p:IsA("BasePart") then p.CollisionGroup = "EnemyOnlyBarrier" end
end)
for _, p in CollectionService:GetTagged("PlayerOnlyBarrier") do p.CollisionGroup = "PlayerOnlyBarrier" end
for _, p in CollectionService:GetTagged("EnemyOnlyBarrier") do p.CollisionGroup = "EnemyOnlyBarrier" end
```

A wall tagged `"PlayerOnlyBarrier"` is **invisible to enemies** (no collision) but **solid to players**. Enemies can spawn behind it, walk through it into the arena; players can't follow back.

For the inverse — a wall enemies can't pass but players can — tag it `"EnemyOnlyBarrier"`.

### Pattern: faction-specific doors

For more groups (Red team, Blue team, Neutral, plus enemies):

```lua
local FACTIONS = {"Red", "Blue", "Neutral", "Enemy"}
local DOOR_GROUPS = {"RedDoor", "BlueDoor"}

for _, f in FACTIONS do PhysicsService:RegisterCollisionGroup(f) end
for _, d in DOOR_GROUPS do PhysicsService:RegisterCollisionGroup(d) end

-- RedDoor: only Red team passes; everyone else blocked
PhysicsService:CollisionGroupSetCollidable("Red", "RedDoor", false)
PhysicsService:CollisionGroupSetCollidable("Blue", "RedDoor", true)
PhysicsService:CollisionGroupSetCollidable("Neutral", "RedDoor", true)
PhysicsService:CollisionGroupSetCollidable("Enemy", "RedDoor", true)

-- Symmetric for BlueDoor
```

When a player joins/changes team, update their character's CollisionGroup:
```lua
player:GetPropertyChangedSignal("Team"):Connect(function()
    if not player.Character then return end
    local groupName = player.Team and player.Team.Name or "Neutral"
    for _, p in player.Character:GetDescendants() do
        if p:IsA("BasePart") then p.CollisionGroup = groupName end
    end
end)
```

### Pattern: phantom / phasing units

A `"Phantom"` collision group that's incollidable with everything except `"Default"` floors:

```lua
PhysicsService:RegisterCollisionGroup("Phantom")
for _, otherGroup in PhysicsService:GetCollisionGroups() do
    if otherGroup.name ~= "Default" then
        PhysicsService:CollisionGroupSetCollidable("Phantom", otherGroup.name, false)
    end
end
-- Phantom still collides with Default (the ground); doesn't collide with players, enemies, walls

-- Apply on spawn:
for _, p in phantomEnemy:GetDescendants() do
    if p:IsA("BasePart") then p.CollisionGroup = "Phantom" end
end
```

Phantom enemies walk through walls and players. Useful for ghosts, ethereal bosses, no-clip ability targets.

### Asymmetric vertical (drop-through floors)

Roblox doesn't natively support "solid from above, pass-through from below" without scripting. Closest implementation: detect the player's vertical velocity in a Heartbeat, set the floor's CollisionGroup based on direction:

```lua
-- For each "Dropthrough" tagged platform:
RunService.Heartbeat:Connect(function()
    for _, p in CollectionService:GetTagged("Dropthrough") do
        local nearest, vel = findNearestPlayerVerticalVelocity(p.Position, 5)
        if vel > 0 then
            -- Player jumping up through it: pass-through
            p.CollisionGroup = "DropthroughOpen"
        else
            -- Player falling onto it: solid
            p.CollisionGroup = "DropthroughSolid"
        end
    end
end)
```

This is fiddly; for most games, just use a one-way ramp or a `CanCollide` toggle gated on the player's S-key (input-driven drop-through).

### Pitfalls (selective barriers)

- **Forgetting to assign the group on character respawn.** `CharacterAdded` re-creates the character with default groups. Re-assign in CharacterAdded.
- **CollisionGroup set on the client.** Doesn't affect server simulation. Always set server-side.
- **Tag set in Studio but no setup script registered.** The barrier looks tagged but has the default collision group. Always pair tag with a CollectionService:GetInstanceAddedSignal.
- **Group names colliding with engine reserved names.** Don't use "Default" as a custom group name.
- **Too many collision groups.** Roblox supports up to 32. With Players + Enemies + 5 teams + 5 door types = many slots. Plan ahead.
- **Phantom units that fall through the floor.** Make sure Phantom still collides with Default (the ground), or they'll fall forever.
- **NoCollisionConstraint vs CollisionGroup confusion.** NoCollisionConstraint is for **one specific pair of parts** (great for "this character's tool doesn't collide with the character"). CollisionGroups are for **categories**.

## Network ownership and collisions

The owner of a part simulates its physics. By default, unanchored parts near a player are owned by that player's client. To keep the server in charge:

```lua
projectile:SetNetworkOwner(nil)        -- server simulates
projectile:SetNetworkOwner(somePlayer) -- specific client simulates
```

For anything gameplay-critical (hit detection, hazards, NPCs), set ownership to `nil`. See [security.md](security.md).

## Patterns

### Trigger volume
```lua
local trigger = Instance.new("Part")
trigger.Anchored = true
trigger.CanCollide = false      -- doesn't block
trigger.CanQuery = false        -- doesn't show in raycasts
trigger.Transparency = 1        -- invisible
trigger.Size = Vector3.new(20, 10, 20)
trigger.Parent = workspace

trigger.Touched:Connect(function(hit)
    local hum = hit.Parent and hit.Parent:FindFirstChildOfClass("Humanoid")
    if hum then onPlayerEnter(hum) end
end)
```

### Per-frame overlap (when `Touched` won't fire because of anchored geometry)
```lua
local op = OverlapParams.new()
op.FilterDescendantsInstances = {trigger}
op.FilterType = Enum.RaycastFilterType.Exclude

RunService.Heartbeat:Connect(function()
    local inside = workspace:GetPartBoundsInBox(trigger.CFrame, trigger.Size, op)
    -- handle entered/left
end)
```

### Debounced damage on touch
```lua
local cooldown = {}
hazard.Touched:Connect(function(hit)
    local hum = hit.Parent and hit.Parent:FindFirstChildOfClass("Humanoid")
    if not hum or cooldown[hum] then return end
    cooldown[hum] = true
    hum:TakeDamage(20)
    task.delay(1, function() cooldown[hum] = nil end)
end)
```

## Pitfalls

- **`Touched` not firing on anchored-anchored contact.** At least one party must be unanchored, OR you handle detection via overlap queries.
- **`Touched` storms.** Fires every physics step while in contact. Always debounce.
- **Forgetting to set CollisionGroup on every descendant** of a Model. Iterate descendants when assigning.
- **Setting CollisionGroups on the client.** Doesn't affect server simulation. Always do group setup on the server.
- **`CanQuery=false` and forgetting later.** Spatial queries silently miss the part forever; debugging is painful.
- **Network ownership confusion.** A part you spawn server-side can quickly transfer to a nearby player's network. Lock it with `SetNetworkOwner(nil)` immediately.

## See also
- [physics-constraints.md](physics-constraints.md) — NoCollisionConstraint
- [raycasting.md](raycasting.md) — overlap queries
- [security.md](security.md) — network ownership
- [parts-and-models.md](parts-and-models.md) — CanCollide/CanQuery/CanTouch as part properties
- [performance.md](performance.md) — CollisionFidelity costs
- [workspace-hierarchy.md](workspace-hierarchy.md) — assemblies and welds
- [world-mechanics.md](world-mechanics.md) — Touched-driven jump pads, trigger volumes, tagged pickups
- [community-libraries.md](community-libraries.md) — **ZonePlus** for ergonomic enter/exit volume detection
