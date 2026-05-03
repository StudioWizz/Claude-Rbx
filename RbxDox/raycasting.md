# Raycasting & Spatial Queries

> Upstream: <https://create.roblox.com/docs/workspace/raycasting>
> Source: `content/en-us/workspace/raycasting.md`

## Purpose
Raycasting fires a virtual line from a point in a direction and reports the first thing it hits. It's the primitive behind hitscan weapons, line-of-sight checks, mouse-to-world picking, and AI vision. Roblox also exposes overlap queries (box / sphere / part) for area lookups.

## Raycast

```lua
local params = RaycastParams.new()
params.FilterType = Enum.RaycastFilterType.Exclude   -- or Include
params.FilterDescendantsInstances = {character}      -- skip these
params.IgnoreWater = false
params.RespectCanCollide = false   -- if true, parts with CanCollide=false are skipped
params.CollisionGroup = "Default"
params.BruteForceAllSlow = false   -- if true, ignores CanQuery — use sparingly

local origin = Vector3.new(0, 5, 0)
local direction = Vector3.new(0, -100, 0)   -- length matters: this caps at 100 studs

local result = workspace:Raycast(origin, direction, params)
if result then
    print(result.Instance, result.Position, result.Normal, result.Distance, result.Material)
end
```

`direction` length is the max distance — there's no separate `length` parameter. To raycast 50 studs forward: `dir = lookVector * 50`.

`RaycastResult` fields: `Instance`, `Position`, `Normal`, `Material`, `Distance`.

## Overlap queries (volume tests)

```lua
local op = OverlapParams.new()
op.FilterType = Enum.RaycastFilterType.Exclude
op.FilterDescendantsInstances = {character}
op.MaxParts = 100
op.CollisionGroup = "Default"
op.RespectCanCollide = false

-- All parts inside a box at cframe with size
local parts = workspace:GetPartBoundsInBox(cframe, size, op)

-- All parts within radius of position
local parts = workspace:GetPartBoundsInRadius(position, radius, op)

-- All parts intersecting a given part's geometry
local parts = workspace:GetPartsInPart(part, op)
```

These use bounding-box tests for `Bounds` variants — fast but not pixel-precise. `GetPartsInPart` does precise intersection.

## Shapecasts

For "swept" volume queries (think: "if I move this box by D, what does it hit first?"):

```lua
local result = workspace:Blockcast(cframe, size, direction, params)
local result = workspace:Spherecast(origin, radius, direction, params)
```
Result shape is the same as Raycast.

## Patterns

### Mouse-to-world (client)
```lua
local UserInputService = game:GetService("UserInputService")
local cam = workspace.CurrentCamera

local function getMouseTarget()
    local mouse = UserInputService:GetMouseLocation()
    local ray = cam:ViewportPointToRay(mouse.X, mouse.Y)
    return workspace:Raycast(ray.Origin, ray.Direction * 1000, params)
end
```

### Line of sight from NPC to player
```lua
local origin = npcHead.Position
local target = playerHRP.Position
local dir = (target - origin)
local result = workspace:Raycast(origin, dir, params)
local visible = result and result.Instance:IsDescendantOf(playerCharacter)
```

### Hitscan weapon (server-side)
```lua
-- Server, after validating the shot from the client
local params = RaycastParams.new()
params.FilterDescendantsInstances = {shooterCharacter}

local hit = workspace:Raycast(origin, direction * MAX_RANGE, params)
if hit then
    local hum = hit.Instance.Parent and hit.Instance.Parent:FindFirstChildOfClass("Humanoid")
    if hum then hum:TakeDamage(damage) end
end
```

## Pitfalls

- **Forgetting `direction` is a length-bearing vector.** `Vector3.new(0, -1, 0)` only travels 1 stud. Multiply by your max range.
- **Self-hits.** Without an Exclude filter, the ray can hit the shooter's own character. Include `character` (or the shooter's part) in `FilterDescendantsInstances`.
- **`RespectCanCollide`.** Default is `false` — `CanCollide=false` parts WILL block your ray unless you set this true. Use `CanQuery=false` on parts you never want hit (decals, effects).
- **`CanQuery=false` parts** are fully invisible to all spatial queries — useful for FX parts, dangerous if you forget you set it.
- **Raycasting from inside a part.** The hit point will be on the inside surface, normal flipped. Offset origin slightly.
- **Querying every frame in a hot loop.** Cache results when possible; raycasts are fast but not free.

## See also
- [collisions.md](collisions.md) — collision groups and CanQuery
- [security.md](security.md) — re-validate hit detection server-side
- [characters.md](characters.md) — Humanoid for damage application
- [cframes-vectors.md](cframes-vectors.md) — direction vectors and CFrame.lookAt for ray construction
- [tools.md](tools.md), [parts-and-models.md](parts-and-models.md) — common callers for hit detection
