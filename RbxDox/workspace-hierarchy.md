# Workspace Hierarchy

> Upstream: <https://create.roblox.com/docs/parts/models> · <https://create.roblox.com/docs/projects/data-model>
> Source: distilled from `parts/models.md`, `projects/data-model.md`, BasePart/Model/Folder API reference

## Purpose
`Workspace` is the root container for everything physically in the 3D world. How you organise the tree underneath it shapes how things move, replicate, get queried, and stay maintainable. Two organising containers — `Folder` and `Model` — plus the rules for what counts as a "child" of what, are the whole story.

For service-level placement (when to use Workspace vs ReplicatedStorage vs ServerStorage vs StarterPlayer/etc.), see [project-structure.md](project-structure.md). This card is about everything *inside* Workspace.

## Folder vs Model — pick the right grouping

Both group children. They behave very differently for movement, physics, and replication.

| | `Folder` | `Model` |
|---|---|---|
| Has a CFrame? | **No** — purely organisational | **Yes** — has a pivot |
| Move all children at once with `:PivotTo`? | No | Yes |
| Counts as a single physics assembly? | No | No (parts inside still simulate independently unless joined) |
| Streams as a unit? | No | Yes (set `ModelStreamingMode`) |
| Has `PrimaryPart`? | No | Yes |
| Auto-cleans on `:Destroy`? | Yes (children too) | Yes (children too) |
| Used for character rigs / vehicles? | No | Yes — characters are Models |
| Visible in physics queries? | Doesn't matter (no geometry) | Doesn't matter (queries hit the BaseParts inside) |

**Rule of thumb:**
- Use a **Folder** when you just want to keep the Explorer tidy. Decorations, lights, ambient parts — anything you'll never want to move as a group.
- Use a **Model** when the children are conceptually one *thing* — a building, a vehicle, a weapon, a character. Anything you might want to clone, teleport, stream, or pivot together.

## What can be a child of what

Five categories of in-world Instance:

1. **Containers** — `Folder`, `Model`, `Actor` (parallel-Luau Model).
2. **Geometry** — `Part`, `MeshPart`, `TrussPart`, `WedgePart`, `CornerWedgePart`, `UnionOperation` (CSG). All inherit from `BasePart`.
3. **Attached visuals/audio** — `Decal`, `Texture`, `SurfaceAppearance`, `MaterialVariant`, `ParticleEmitter`, `Trail`, `Beam`, `PointLight`/`SurfaceLight`/`SpotLight`, `Sound`, `Smoke`/`Fire`/`Sparkles`, `Highlight`, `SelectionBox`. These attach to a parent BasePart and ride with it.
4. **Constraints / joints** — `Attachment`, `WeldConstraint`, `HingeConstraint`, `Motor6D`, etc. Parented inside a BasePart; the constraint references two parts via attachments.
5. **Logic-bearing** — `Script` (with `RunContext = Server`), `LocalScript` (only meaningful under a Player), `ModuleScript`, `BindableEvent`, `RemoteEvent`, `ProximityPrompt`, `ClickDetector`, `Animation`, `AnimationController`.

Roughly:
- **Folders/Models contain** other Folders/Models and BaseParts.
- **BaseParts contain** attached visuals/audio, constraints, and gameplay logic objects (`ProximityPrompt`, `ClickDetector`, `Sound`, `Script`).
- **Cross-tree references** go through Attachments (constraints) or `ObjectValue` instances, not parenting.

## Parenting semantics

`instance.Parent = newParent` is the single most consequential operation in the engine.

- The first time an Instance becomes a descendant of `game`, **replication kicks in** (the engine starts streaming the Instance to all relevant clients).
- Reparenting a part inside Workspace **moves it in the tree**, not in space — `Position`/`CFrame` stay the same.
- Setting `Parent = nil` removes it from the world but **doesn't disconnect connections**. Use `:Destroy()` to fully clean up — `Parent = nil` followed by losing the reference will eventually GC, but any signal subscriber keeps it alive.
- Reparenting an unanchored part inside an existing assembly can **break/rejoin welds** depending on what holds them together (WeldConstraint vs old Weld).

## Pivot, PrimaryPart, and moving Models

A Model has a `Pivot` — a CFrame used as the move-anchor for `:PivotTo(target)` and `:GetPivot()`. The pivot is independent of any specific child part by default; you can set it manually in Studio (Model tab → Edit Pivot).

`Model.PrimaryPart` is an optional pointer to a "main" BasePart; some legacy APIs (`SetPrimaryPartCFrame`, deprecated) rely on it. Modern code:

```lua
model:PivotTo(CFrame.new(0, 50, 0))     -- moves whole model so pivot lands at target
local pivot = model:GetPivot()
local size = model:GetExtentsSize()      -- bounding box size
local cf, sz = model:GetBoundingBox()    -- oriented bounding box
```

`PivotTo` preserves the relative layout of every child — including Attachments, decorations, and welded sub-models. It's the right way to teleport a vehicle, a building, or a whole assembled structure.

## Build-then-parent

The single most important perf habit when constructing things at runtime:

```lua
-- BAD: each Parent triggers replication + physics setup
for i = 1, 100 do
    local p = Instance.new("Part")
    p.Parent = workspace
    p.Position = Vector3.new(i, 5, 0)
end

-- GOOD: build the tree detached, then parent the root once
local m = Instance.new("Model")
for i = 1, 100 do
    local p = Instance.new("Part")
    p.Position = Vector3.new(i, 5, 0)
    p.Parent = m
end
m.Parent = workspace      -- single replication step
```

Same applies to UI under a ScreenGui, Folders of replicated assets, etc.

## Terrain

`Workspace.Terrain` is a singleton child of Workspace, present in every place. It holds voxel terrain (smooth, with materials) — separate from BaseParts. Don't try to delete or reparent it. Edit via the Terrain Editor or the `Terrain` API:

```lua
workspace.Terrain:FillBlock(cframe, size, Enum.Material.Grass)
workspace.Terrain:FillBall(position, radius, Enum.Material.Rock)
workspace.Terrain:Clear()
```

Terrain has its own collision and rendering pipeline; it's cheaper than equivalent geometry made of parts but not free. It doesn't take attached children (no decals on terrain).

## WorldModel (advanced)

`WorldModel` is a Model subclass that supports physics queries on its descendants even when it's NOT in Workspace — useful for ViewportFrame previews or off-world simulation:

```lua
local wm = Instance.new("WorldModel")
wm.Parent = viewportFrame
-- parts under wm participate in :Raycast etc., independently of Workspace
```

Niche. You only need this for ViewportFrame interactivity or sandboxed physics.

## Naming and depth

- Names don't have to be unique among siblings, but **they should be**, or `:FindFirstChild`/`WaitForChild` becomes ambiguous.
- Avoid spaces and punctuation in names you'll index by (`folder.MyModel` vs `folder["My Model"]`).
- Keep depth shallow. A 10-level-deep Model tree slows `GetDescendants` and complicates `WaitForChild` chains.
- Conventional top-level Workspace structure for a non-trivial place:
```
Workspace
├── Terrain
├── Camera          (the active CurrentCamera)
├── Baseplate       (or Map model)
├── Map/            (Folder containing static decor)
├── NPCs/           (Folder of NPC Models)
├── Spawns/         (Folder of SpawnLocations)
├── Effects/        (Folder for runtime-created FX)
└── Debug/          (optional, devs only)
```

Player Characters appear here too as Models when they spawn (parented directly under Workspace).

## Patterns

### Move a building without breaking it
```lua
building:PivotTo(CFrame.new(newOrigin) * (building:GetPivot() - building:GetPivot().Position))
-- preserves rotation; just translates pivot
```

### Clone an asset and place it
```lua
local template = ReplicatedStorage.Templates.Tower
local clone = template:Clone()
clone:PivotTo(CFrame.new(spawnPoint.Position))
clone.Parent = workspace.Map
```

### Sweep a Folder of NPCs
```lua
for _, npc in workspace.NPCs:GetChildren() do
    if npc:IsA("Model") and npc:FindFirstChildOfClass("Humanoid") then
        update(npc)
    end
end
```

### Iterate descendants of a tag (cheaper than walking Workspace)
```lua
local CollectionService = game:GetService("CollectionService")
for _, hazard in CollectionService:GetTagged("Hazard") do
    -- the tag finds them no matter where in the tree they sit
end
```
See [services.md](services.md) and [attributes.md](attributes.md) for the tag-driven pattern.

## Pitfalls

- **Using a Folder when you wanted a Model.** You can't `:PivotTo` a Folder — moving 50 children individually is slow and error-prone. Convert to a Model.
- **Using a Model when you wanted a Folder.** Models compute bounding boxes and pivots; over-using them on purely-decorative groups wastes a tiny amount of work and clutters the API surface.
- **Parenting parts directly to Workspace at runtime in a loop.** Build inside a detached parent, then assign `Parent` once — see "Build-then-parent" above.
- **Setting `Parent = nil` instead of `:Destroy()`.** Connections stay alive; the Instance is held alive by them. Always `:Destroy()` for cleanup.
- **Renaming a Model whose `PrimaryPart` references are hard-coded.** Hard-coded paths break silently. Prefer `:FindFirstChildOfClass` or tags.
- **Putting `LocalScript` under Workspace.** Won't run. LocalScripts live under a Player container or inside StarterCharacter.
- **Reparenting a Character.** The engine assumes player Characters are direct children of Workspace; moving them under a Folder breaks Humanoid behaviour and replication.
- **Deeply nested Models for organisation.** Pure organisation belongs in Folders. Reserve nesting for actual physical sub-assemblies.
- **Forgetting `WaitForChild` after spawning a Model on the server.** On the client, the Model may arrive a frame before its children. Wait, or use `Model.ModelStreamingMode` to control atomicity.

## See also
- [project-structure.md](project-structure.md) — service-level placement (Workspace vs ReplicatedStorage etc.)
- [parts-and-models.md](parts-and-models.md) — BasePart and Model APIs in depth
- [characters.md](characters.md) — the specific Character Model layout
- [collisions.md](collisions.md) — how parenting interacts with assemblies and welds
- [performance.md](performance.md) — Streaming, build-then-parent, GetDescendants cost
- [areas-and-biomes.md](areas-and-biomes.md) — areas as a structural concept built on top of Folder/Model organization
- [procedural-construction.md](procedural-construction.md) — script-driven generation of structures using these organisational rules
