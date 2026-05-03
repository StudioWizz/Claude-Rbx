# Parts & Models

> Upstream: <https://create.roblox.com/docs/parts>
> Source: `content/en-us/parts/index.md`, `models.md`, `materials.md`, `meshes.md`, `textures-decals.md`

## Purpose
A **Part** is the basic 3D primitive (block, ball, cylinder, wedge). A **Model** is a container that groups multiple parts so they move and replicate together. **MeshPart** is a Part with a custom mesh. These three building blocks make up basically everything you'll see in a Roblox world.

## BasePart key properties
All parts (Part, MeshPart, TrussPart, etc.) inherit from `BasePart`:

- `Name`, `Parent`
- `Size: Vector3`, `Position: Vector3`, `Orientation: Vector3 (degrees)`, `CFrame: CFrame` (use CFrame for rotation work)
- `Anchored: bool` — if true, ignored by physics; otherwise simulated
- `CanCollide: bool` — does it physically collide with others
- `CanQuery: bool` — does it appear in raycasts / `:GetPartBoundsIn*`
- `CanTouch: bool` — does it fire `Touched` / `TouchEnded`
- `Massless: bool` — contributes 0 mass to its assembly
- `Material: Enum.Material` — `Plastic`, `Wood`, `Concrete`, `Neon`, `Glass`, `ForceField`, `Sand`, `Marble`, `Ice`, etc.
- `Color: Color3` (preferred) or `BrickColor: BrickColor` (legacy enum)
- `Transparency: number` 0–1
- `Reflectance: number` 0–1
- `Shape: Enum.PartType` — `Block`, `Ball`, `Cylinder`, `Wedge`, `CornerWedge` (Part only)
- `CollisionGroup: string` — see [collisions.md](collisions.md)
- `CustomPhysicalProperties: PhysicalProperties` — override density/friction/elasticity per-part
- `AssemblyLinearVelocity`, `AssemblyAngularVelocity` — read/write the whole rigid assembly
- `Pivot: CFrame` and `:PivotTo(cf: CFrame)` — pivot is the move-anchor; `PivotTo` keeps a Model's relative layout

## Models
A `Model` groups parts. Key API:

- `model.PrimaryPart` — the canonical "anchor" part (set this manually)
- `model:GetPivot()` — the model's pivot CFrame
- `model:PivotTo(cf)` — move the whole model so its pivot lands at `cf` (preserves internal layout). **Use this instead of `:SetPrimaryPartCFrame`**, which is deprecated.
- `model:GetExtentsSize()` — bounding-box size
- `model:GetBoundingBox()` — `(CFrame, Vector3)` of the OBB
- `model:GetDescendants()`, `model:GetChildren()`

A model can also be a `WorldModel` (allows physics queries on a model not in `Workspace`) — niche.

## MeshPart
A `MeshPart` references a mesh asset (`MeshId = "rbxassetid://..."`) and optional texture (`TextureID`). Created by importing a `.fbx`/`.obj` via the Asset Manager. Uses one collision representation:
- `CollisionFidelity = Default | Hull | PreciseConvexDecomposition | Box`
- `RenderFidelity = Automatic | Precise | Performance`

Higher fidelities cost more — pick the lowest that looks/behaves right.

## Materials
- **Built-in materials**: change the look (Plastic, Wood, Marble, Neon for emissive, ForceField for shimmer, Glass for transmission, etc.).
- **MaterialVariant**: custom PBR material set you author or generate; assign by `Part.MaterialVariant = "MyVariant"`.
- **SurfaceAppearance**: child of a MeshPart that overrides its texturing with full PBR (Color, Normal, Metalness, Roughness maps).

## Textures & Decals
- `Decal` — a child of a Part, applies an image to one face. Set `Texture = "rbxassetid://..."`, `Face = Enum.NormalId.Front`.
- `Texture` — like Decal but tiles based on `StudsPerTileU/V`.

## Patterns

### Spawn a part
```lua
local part = Instance.new("Part")
part.Size = Vector3.new(4, 1, 4)
part.Material = Enum.Material.Wood
part.Color = Color3.fromRGB(120, 80, 40)
part.Anchored = true
part.CFrame = CFrame.new(0, 5, 0)
part.Parent = workspace
```

### Build then parent (one assembly add, not many)
```lua
local m = Instance.new("Model")
for i = 1, 100 do
    local p = Instance.new("Part")
    p.Parent = m       -- cheap: m isn't in Workspace yet
end
m.Parent = workspace   -- single replication step
```
Adding to Workspace one-at-a-time triggers per-Instance replication and physics setup. Build inside a detached parent first.

### Move a model
```lua
model:PivotTo(CFrame.new(targetPos))
```

### Cleanup
```lua
local Debris = game:GetService("Debris")
Debris:AddItem(part, 5)   -- destroy in 5 seconds
-- or explicitly:
part:Destroy()
```
`Destroy` disconnects all signals and removes the Instance. Always call it on cleanup; setting `Parent = nil` alone leaks references.

## Pitfalls

- **`Anchored` parts ignore physics** — they won't fall, they don't have meaningful mass. CFrame them to move.
- **Setting `Position` rotates nothing.** `Position` is a shorthand for `CFrame.Position`. To both move and rotate, use `CFrame`.
- **`CFrame` ≠ `Position`** when the part is part of a welded assembly — the engine moves the whole assembly. For models, use `:PivotTo`.
- **Per-frame `Instance.new`.** Instancing is cheap-ish but parenting to Workspace is expensive. Pool/reuse for projectiles, particles, etc.
- **Forgetting `:Destroy()`.** Reparenting to nil leaves event connections alive — slow leak.
- **High-poly meshes with `Precise` collision fidelity** can wreck physics performance. Use `Hull` or `Box` unless precise is required.

## See also
- [workspace-hierarchy.md](workspace-hierarchy.md) — Folder vs Model, parenting semantics, pivot, build-then-parent
- [procedural-construction.md](procedural-construction.md) — Claude/script-driven construction patterns built on these primitives
- [cframes-vectors.md](cframes-vectors.md) — moving and rotating
- [collisions.md](collisions.md) — collision groups, CanCollide
- [raycasting.md](raycasting.md) — spatial queries
- [vfx.md](vfx.md) — ParticleEmitter / Beam / Trail parented under parts
- [lighting.md](lighting.md) — `Lighting.Technology` affects how materials render
