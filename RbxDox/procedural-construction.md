# Procedural Construction (Claude Building From Parts)

> Source: distilled from realistic patterns for AI-driven construction in Roblox via the MCP

## Purpose
This card is about **what Claude can reasonably build from `BasePart` primitives**, the patterns for doing so via the Roblox Studio MCP, and the alternatives (Creator Store assets, generated meshes) that are usually a better fit for organic/aesthetic objects. `parts-and-models.md` covers the primitives; `workspace-hierarchy.md` covers organisation; `studio-mcp.md` lists the MCP tools. This card is the construction *workflow* — the patterns Claude uses to assemble things, the decision tree for when to build vs insert vs generate, the iteration loop with the user via screen captures, and explicit limits so we don't try things that won't work.

## Realistic limits — what Claude can and can't build well

### Good fit (procedural / pattern-driven)
- Geometric structures: towers, walls, floors, ceilings, grids, stairs, arches, columns
- Repeated patterns: fence rows, picket arrays, brick walls, tiled floors
- Parametric variations: "make this 3 stories tall", "replace all walls with stone"
- Algorithmic organics with good defaults: L-system trees, voronoi rocks, perlin terrain bumps, simple shrubs
- Modifications to existing structures: "tint everything wood-colored", "add windows every 4 studs"
- Layouts that follow a spec: "build a 5×5 dungeon room with a door on the north wall"

### Poor fit (free-form aesthetic)
- "Make a beautiful Victorian mansion" — no clear spec; Claude has no visual feedback to judge "beautiful"
- High-fidelity decorative props (a realistic-looking lamp post; a detailed sword)
- Anything where the answer is "it should look right" without measurable criteria
- Iterating on aesthetics by feel alone — Claude can build, but can't tell if the result is good without screen captures and the user judging

### When in doubt
Don't try to build a tree from 200 wedge parts. Use:
1. **Creator Store** — `mcp__Roblox_Studio__search_creator_store("oak tree")` then `insert_from_creator_store(assetId)` — instant, looks great, no Claude effort
2. **`generate_mesh`** — Roblox's AI mesh generator. Costs and quality vary. Good for one-off shapes that don't exist on the Creator Store.
3. **`generate_material`** — for texture variation on parts you've built

## Decision tree

```
Building <thing>?
│
├── Is it geometric / repeating / pattern-based?
│   └── YES → Procedural script (this card)
│
├── Does it exist on the Creator Store?
│   └── YES → search_creator_store + insert_from_creator_store
│
├── Is it organic but follows known algorithm (tree, rocks, terrain)?
│   └── YES → Algorithmic procedural (this card, "Algorithmic organics" section)
│
├── Is it a one-off custom shape that doesn't exist?
│   └── MAYBE → generate_mesh (preview before committing)
│
└── Is it "make it look nice"?
    └── Ask the user for: reference image, size, color palette, style — convert to spec first
```

The cheapest correct answer is almost always: **insert an existing asset**. Roblox has thousands of free trees, lamp posts, furniture pieces. Use them.

## Build-then-parent (recap)

The single most important perf pattern, restated here because every construction script touches it:

```lua
-- BAD: parents to Workspace one at a time → N replication steps
for i = 1, 100 do
    local p = Instance.new("Part")
    p.Position = Vector3.new(i, 5, 0)
    p.Parent = workspace
end

-- GOOD: build inside a detached Model, parent once at the end
local m = Instance.new("Model")
m.Name = "GeneratedFence"
for i = 1, 100 do
    local p = Instance.new("Part")
    p.Position = Vector3.new(i, 5, 0)
    p.Anchored = true
    p.Parent = m
end
m.Parent = workspace.GeneratedStructures        -- single replication step
```

For everything below, assume parts are anchored, `CanCollide = true` for structural geometry / `false` for decorative, and parented into a detached Model that gets parented to Workspace at the end. See [workspace-hierarchy.md](workspace-hierarchy.md).

## Procedural primitives — pattern library

### Wall
```lua
local function buildWall(origin: CFrame, length: number, height: number, thickness: number, material: Enum.Material): Model
    local m = Instance.new("Model"); m.Name = "Wall"
    local p = Instance.new("Part")
    p.Anchored = true
    p.Material = material
    p.Size = Vector3.new(length, height, thickness)
    p.CFrame = origin * CFrame.new(0, height/2, 0)
    p.Parent = m
    return m
end
```

For walls with windows / doors, build as several parts subtracted-by-script (multiple BaseParts forming the silhouette), or use solid modeling (`UnionAsync`).

### Floor / ceiling
```lua
local function buildFloor(origin: CFrame, width: number, depth: number, thickness: number?): Model
    thickness = thickness or 0.5
    local m = Instance.new("Model"); m.Name = "Floor"
    local p = Instance.new("Part")
    p.Anchored = true
    p.Size = Vector3.new(width, thickness, depth)
    p.CFrame = origin * CFrame.new(0, -thickness/2, 0)
    p.Parent = m
    return m
end
```

### Stairs (straight)
```lua
local function buildStairs(origin: CFrame, stepCount: number, stepRise: number, stepRun: number, width: number): Model
    local m = Instance.new("Model"); m.Name = "Stairs"
    for i = 1, stepCount do
        local p = Instance.new("Part")
        p.Anchored = true
        p.Size = Vector3.new(width, stepRise, stepRun)
        p.CFrame = origin * CFrame.new(0, (i - 0.5) * stepRise, -(i - 0.5) * stepRun)
        p.Parent = m
    end
    return m
end
```

### Stairs (spiral)
```lua
local function buildSpiralStairs(center: Vector3, radius: number, stepCount: number, totalHeight: number, totalAngle: number): Model
    local m = Instance.new("Model"); m.Name = "SpiralStairs"
    local stepHeight = totalHeight / stepCount
    for i = 1, stepCount do
        local angle = (i - 1) / stepCount * totalAngle
        local pos = center + Vector3.new(math.cos(angle) * radius, (i - 0.5) * stepHeight, math.sin(angle) * radius)
        local p = Instance.new("Part")
        p.Anchored = true
        p.Size = Vector3.new(radius * 0.8, stepHeight, 2)
        p.CFrame = CFrame.new(pos) * CFrame.Angles(0, -angle, 0)
        p.Parent = m
    end
    return m
end
```

### Tower (vertical stack)
```lua
local function buildTower(base: CFrame, height: number, sides: number, radius: number): Model
    local m = Instance.new("Model"); m.Name = "Tower"
    local segHeight = 4
    local segments = math.ceil(height / segHeight)
    for h = 0, segments - 1 do
        for s = 0, sides - 1 do
            local angle = s / sides * math.pi * 2
            local pos = base.Position + Vector3.new(math.cos(angle) * radius, h * segHeight + segHeight/2, math.sin(angle) * radius)
            local p = Instance.new("Part")
            p.Anchored = true
            p.Size = Vector3.new(2, segHeight, radius * 1.2)
            p.CFrame = CFrame.new(pos) * CFrame.Angles(0, -angle, 0)
            p.Material = Enum.Material.Cobblestone
            p.Parent = m
        end
    end
    return m
end
```

### Arch (segmented)
```lua
local function buildArch(center: Vector3, radius: number, thickness: number, segments: number): Model
    local m = Instance.new("Model"); m.Name = "Arch"
    for i = 0, segments - 1 do
        local angle = (i / (segments - 1)) * math.pi
        local pos = center + Vector3.new(math.cos(math.pi - angle) * radius, math.sin(angle) * radius, 0)
        local segLength = (math.pi * radius) / segments
        local p = Instance.new("Part")
        p.Anchored = true
        p.Size = Vector3.new(thickness, thickness, segLength * 1.05)
        p.CFrame = CFrame.new(pos) * CFrame.Angles(0, math.pi/2, math.pi - angle - math.pi/2)
        p.Parent = m
    end
    return m
end
```

### Fence row
```lua
local function buildFence(start: Vector3, finish: Vector3, postSpacing: number): Model
    local m = Instance.new("Model"); m.Name = "Fence"
    local dir = (finish - start)
    local count = math.floor(dir.Magnitude / postSpacing)
    for i = 0, count do
        local pos = start + dir.Unit * (i * postSpacing)
        local post = Instance.new("Part")
        post.Anchored = true
        post.Size = Vector3.new(0.4, 4, 0.4)
        post.Material = Enum.Material.Wood
        post.CFrame = CFrame.new(pos + Vector3.new(0, 2, 0))
        post.Parent = m
    end
    -- Add cross-rails between posts (omitted; same pattern with horizontal parts)
    return m
end
```

## Compound structures

### Simple house (composition of primitives)
```lua
local function buildHouse(origin: CFrame, width: number, depth: number, height: number): Model
    local house = Instance.new("Model"); house.Name = "House"

    -- Floor
    buildFloor(origin, width, depth).Parent = house
    -- Four walls
    local wallThickness = 0.5
    buildWall(origin * CFrame.new(0, 0, -depth/2), width, height, wallThickness).Parent = house
    buildWall(origin * CFrame.new(0, 0, depth/2),  width, height, wallThickness).Parent = house
    buildWall(origin * CFrame.new(-width/2, 0, 0) * CFrame.Angles(0, math.pi/2, 0), depth, height, wallThickness).Parent = house
    buildWall(origin * CFrame.new(width/2, 0, 0)  * CFrame.Angles(0, math.pi/2, 0), depth, height, wallThickness).Parent = house
    -- Roof (flat or pitched — flat shown; pitched requires wedges)
    local roof = Instance.new("Part")
    roof.Anchored = true
    roof.Size = Vector3.new(width + 1, 0.5, depth + 1)
    roof.CFrame = origin * CFrame.new(0, height + 0.25, 0)
    roof.Parent = house

    return house
end
```

For windows, doors, pitched roofs, dormers — extend by composition. Each new feature is a function that takes a CFrame relative to the building and adds parts.

### Bridge
Span between two points; segment into deck planks; add railings on each side. Parametric in length, width, plank thickness.

### Crenellated tower top
Stack a row of small parts with gaps around a tower's top to give it a castle silhouette.

## Algorithmic organics

These are the procedural alternatives to "use a tree asset." Generally **try the asset first** — these are for when you specifically want generated variation.

### L-system tree
A simple recursive branching tree. Each branch spawns child branches at random angles, shrinking at each level.

```lua
local function makeBranch(origin: CFrame, length: number, thickness: number): Part
    local p = Instance.new("Part")
    p.Anchored = true
    p.Material = Enum.Material.Wood
    p.BrickColor = BrickColor.new("Brown")
    p.Size = Vector3.new(thickness, length, thickness)
    p.CFrame = origin * CFrame.new(0, length/2, 0)
    return p
end

local function makeLeaves(origin: CFrame): Part
    local p = Instance.new("Part")
    p.Anchored = true
    p.Shape = Enum.PartType.Ball
    p.Material = Enum.Material.Grass
    p.BrickColor = BrickColor.new("Forest green")
    p.Size = Vector3.new(3, 3, 3) + Vector3.one * (math.random() * 1.5)
    p.CFrame = origin
    return p
end

local function buildTree(origin: CFrame, depth: number?, length: number?, thickness: number?, parent: Instance?): Model
    depth = depth or 4
    length = length or 6
    thickness = thickness or 1.5
    parent = parent or Instance.new("Model")
    if not parent:IsA("Model") then error("parent must be Model") end
    parent.Name = "Tree"

    if depth <= 0 or length < 0.5 then
        makeLeaves(origin * CFrame.new(0, length/2, 0)).Parent = parent
        return parent
    end

    makeBranch(origin, length, thickness).Parent = parent

    -- Spawn 2-3 child branches at the top, angled outward
    local children = math.random(2, 3)
    for _ = 1, children do
        local pitch = math.rad(20 + math.random() * 30)     -- 20-50° outward
        local yaw = math.random() * math.pi * 2              -- random around
        local childOrigin = origin
            * CFrame.new(0, length, 0)
            * CFrame.Angles(0, yaw, 0)
            * CFrame.Angles(pitch, 0, 0)
        buildTree(childOrigin, depth - 1, length * 0.7, thickness * 0.7, parent)
    end
    return parent
end

-- Use:
-- local tree = buildTree(CFrame.new(0, 0, 0))
-- tree.Parent = workspace
```

Parameters to expose: `depth`, `length`, `thickness`, branch count range, branch angle range, leaf style. Tweak per tree variety (oak = wider angles, pine = narrower + denser).

This builds a low-poly tree. **It will not look as good as a Creator Store asset** — but it's procedural, so you can generate forests of unique variations.

### Bush / shrub
A cluster of slightly-overlapping spheres in random sizes:
```lua
local function buildBush(center: Vector3, radius: number, count: number?): Model
    count = count or 7
    local m = Instance.new("Model"); m.Name = "Bush"
    for i = 1, count do
        local offset = Vector3.new(
            (math.random() * 2 - 1) * radius * 0.6,
            (math.random() * 0.5) * radius * 0.5,
            (math.random() * 2 - 1) * radius * 0.6
        )
        local p = Instance.new("Part")
        p.Shape = Enum.PartType.Ball
        p.Anchored = true
        p.Material = Enum.Material.Grass
        p.BrickColor = BrickColor.new("Forest green")
        p.Size = Vector3.one * (radius * (0.6 + math.random() * 0.4))
        p.Position = center + offset
        p.Parent = m
    end
    return m
end
```

### Rock cluster
Same shape as a bush but with stone material and random rotation:
```lua
local function buildRockCluster(center: Vector3, radius: number, count: number?): Model
    count = count or 5
    local m = Instance.new("Model"); m.Name = "Rocks"
    for i = 1, count do
        local p = Instance.new("Part")
        p.Anchored = true
        p.Material = Enum.Material.Rock
        p.BrickColor = BrickColor.new("Medium stone grey")
        p.Size = Vector3.new(radius, radius * 0.8, radius) + Vector3.one * (math.random() - 0.5) * radius
        p.CFrame = CFrame.new(center + Vector3.new((math.random() * 2 - 1) * radius, 0, (math.random() * 2 - 1) * radius))
            * CFrame.Angles(math.random() * math.pi * 2, math.random() * math.pi * 2, math.random() * math.pi * 2)
        p.Parent = m
    end
    return m
end
```

### Path (winding)
A chain of parts following a noisy line between two points. Useful for forest trails, rivers (with water material), etc.

```lua
local function buildPath(start: Vector3, finish: Vector3, segmentLength: number, width: number): Model
    local m = Instance.new("Model"); m.Name = "Path"
    local dir = (finish - start)
    local count = math.floor(dir.Magnitude / segmentLength)
    local perp = Vector3.new(-dir.Unit.Z, 0, dir.Unit.X)        -- 2D perpendicular

    for i = 0, count do
        local t = i / count
        local jitter = math.sin(t * 8) * width * 0.5             -- snake pattern
        local pos = start:Lerp(finish, t) + perp * jitter
        local p = Instance.new("Part")
        p.Anchored = true
        p.Size = Vector3.new(width, 0.2, segmentLength)
        p.Material = Enum.Material.Slate
        p.BrickColor = BrickColor.new("Brown")
        p.CFrame = CFrame.lookAt(pos, pos + dir.Unit)
        p.Parent = m
    end
    return m
end
```

For rivers, swap the material to `Glass` or use Terrain water for proper buoyancy.

## Iteration workflow with the user

Construction is iterative. The pattern that works:

1. **Get the spec** — Ask: dimensions, materials, color palette, style references. Convert vague ("a little house") to concrete (`width = 16, depth = 12, height = 10, material = Wood, roof = pitched`).
2. **Build a parameterized version** — Write a function with the spec as parameters. Don't hard-code positions; expose them.
3. **Build at a reachable location** — Use a known origin (e.g., `Vector3.new(0, 5, 0)`) so you can find it.
4. **Screen-capture** — `mcp__Roblox_Studio__screen_capture()` after building. The user sees what you made.
5. **Adjust parameters, not code** — "Roof too low" → re-call with `height = 14`. The function stays the same; the call changes.
6. **Iterate visibly** — Each iteration takes <30 seconds. Show the user; ask if it's right.
7. **Commit when approved** — Move from the temporary build location to its final location with `:PivotTo`. Save as a `.rbxm` template if it's reusable.

**Do not** try to nail it on the first attempt. Build → look → adjust. Three iterations usually beats one perfect attempt.

## MCP execute_luau patterns for construction

Run construction code via the MCP. Build into a target Model, parent at the end:

```lua
-- The Luau you'd send via mcp__Roblox_Studio__execute_luau
local function buildCastle()
    local castle = Instance.new("Model")
    castle.Name = "Castle"

    -- Use the function library you've defined (you'd inline these for execute_luau or require from a ModuleScript)
    local function buildTower(...) ... end    -- omitted; same as above

    -- Compose
    buildTower(CFrame.new(-20, 0, -20), 30, 8, 5).Parent = castle
    buildTower(CFrame.new(20, 0, -20), 30, 8, 5).Parent = castle
    buildTower(CFrame.new(-20, 0, 20), 30, 8, 5).Parent = castle
    buildTower(CFrame.new(20, 0, 20), 30, 8, 5).Parent = castle
    -- ... walls between towers, gate, etc.

    castle.Parent = workspace.Generated or workspace
    return castle:GetFullName()
end

return buildCastle()
```

For larger scripts, save the function library as a ModuleScript in `ServerStorage` first via the MCP, then `require` it in subsequent `execute_luau` calls. Keeps individual MCP calls small.

For state inspection between iterations: `mcp__Roblox_Studio__inspect_instance("game.Workspace.Generated.Castle")` to count parts, check sizes, etc.

## Saving structures as data

Once a structure is "right," serialize it for re-use. A template Model can be exported with the Studio's "Save to File" (`.rbxm`) — but for parametric structures, save the **parameters** instead:

```lua
local houseSpec = {
    type = "House",
    width = 16, depth = 12, height = 10,
    material = "Wood",
    roofStyle = "Flat",
    color = Color3.fromRGB(160, 110, 60),
}

-- Persist via DataStore (see datastores.md), or commit to a Lua table in source.
-- Re-build at any point by calling buildHouse(origin, houseSpec).
```

For player-driven sandbox saves, see [placement.md](placement.md) — same idea, different audience.

## Pitfalls

- **Trying to build things Claude shouldn't.** A photoreal lamp post from 50 parts is wasted effort. Use the Creator Store. Recognise the limit.
- **Forgetting `Anchored = true`.** Decorative geometry that's unanchored gets simulated, falls, drifts. Always anchor unless the part is meant to be physical.
- **Per-part Parent to Workspace.** Slow + chatty. Build then parent.
- **Generated Model with no PrimaryPart.** Can't `:PivotTo` reliably. Set `model.PrimaryPart` (often the floor or center part) before parenting.
- **No origin parameter.** Hard-coded `Vector3.new(0, 5, 0)` everywhere means you can't move the structure. Always take an `origin: CFrame` and offset everything from it.
- **Material choices that don't render with current Lighting.Technology.** Set `Lighting.Technology = Future` for full PBR (see [lighting.md](lighting.md)) — Voxel doesn't render SurfaceAppearance.
- **Trees / bushes with hundreds of parts each.** Performance dies fast (see [performance.md](performance.md)). Keep procedural organics low-poly, OR use mesh assets.
- **Iterating without screen captures.** You're flying blind. After every build, `screen_capture` and show the user.
- **Building into Workspace root without organisation.** Random parts everywhere; nothing's findable. Always build into a named Model or Folder under a known structure (e.g., `workspace.Generated`).
- **Forgetting the user's coordinate convention.** They might mean "10 studs north" relative to spawn, not world (0, 0, 10). Confirm direction conventions before building.
- **Generating thousands of parts in a single `execute_luau` call.** The call yields and may time out. Chunk into batches; use `task.wait()` between chunks.
- **`UnionAsync` / `SubtractAsync` overuse.** They're slow and can fail. For visual subtraction, build the silhouette out of multiple BaseParts instead.
- **No cleanup before re-iteration.** Each iteration adds another castle next to the previous one. `castle:Destroy()` before re-building, or use a fixed name and check if it exists.
- **Hardcoded asset IDs that don't exist in the user's experience.** Asset IDs are global but ownership/permission may not allow them in this place. Test before committing.

## See also
- [parts-and-models.md](parts-and-models.md) — `BasePart` properties, `Model:PivotTo`
- [workspace-hierarchy.md](workspace-hierarchy.md) — Folder vs Model organization, build-then-parent (foundational)
- [studio-mcp.md](studio-mcp.md) — `execute_luau`, `screen_capture`, `inspect_instance`, Creator Store tools
- [cframes-vectors.md](cframes-vectors.md) — CFrame composition for positioning generated parts
- [placement.md](placement.md) — player-driven placement (this card is Claude/script-driven; same primitive, different driver)
- [lighting.md](lighting.md) — Lighting.Technology = Future for proper material rendering
- [performance.md](performance.md) — part count, collision fidelity, perf for large generated structures
- [community-libraries.md](community-libraries.md) — Creator Store assets are usually a better answer than rebuilding from parts
- [datastores.md](datastores.md) — persisting parametric build specs
- [areas-and-biomes.md](areas-and-biomes.md) — generated areas (procgen dungeons, biome population)
- [vfx.md](vfx.md) — particle / emitter decoration on generated structures
- [growth-systems.md](growth-systems.md) — applied use of the L-system pattern for plant growth across stages
