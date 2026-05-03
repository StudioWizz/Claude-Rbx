# CFrames & Vectors

> Upstream: <https://create.roblox.com/docs/workspace/cframes>
> Source: `content/en-us/workspace/cframes.md`

## Purpose
A **`Vector3`** is a 3D position or direction. A **`CFrame`** ("coordinate frame") is a position **and** an orientation — Roblox's combined translation + rotation matrix. Anything that has a CFrame property defines its placement in 3D space with one value.

## Vector3
```lua
local v = Vector3.new(1, 2, 3)
v.X; v.Y; v.Z
v.Magnitude               -- length
v.Unit                    -- normalized (magnitude 1, or NaN if zero)

-- Operators
local sum = v1 + v2
local diff = v1 - v2
local scaled = v * 2
local dot = v1:Dot(v2)
local cross = v1:Cross(v2)
local lerped = v1:Lerp(v2, 0.5)

-- Common constants
Vector3.zero
Vector3.one
Vector3.xAxis  -- (1,0,0)
Vector3.yAxis  -- (0,1,0)
Vector3.zAxis  -- (0,0,1)
```

Roblox's coordinate system: **+Y is up**, **-Z is forward** (the camera looks down -Z by default). Studs are the unit; 1 stud ≈ 28 cm.

## CFrame

A CFrame stores: position (a Vector3) + a 3×3 rotation matrix (X, Y, Z basis vectors).

### Construction
```lua
CFrame.new()                          -- identity at origin
CFrame.new(0, 5, 0)                   -- translation only
CFrame.new(pos: Vector3)              -- translation
CFrame.new(eye, target)               -- look from eye toward target (-Z faces target)
CFrame.lookAt(eye, target, up?)       -- preferred modern form
CFrame.fromEulerAnglesXYZ(rx, ry, rz) -- radians
CFrame.Angles(rx, ry, rz)             -- alias for above
CFrame.fromAxisAngle(axis, angle)
CFrame.fromOrientation(rx, ry, rz)    -- YXZ order, matches Studio's Orientation
```

### Components
```lua
cf.Position                 -- Vector3
cf.LookVector               -- forward (-Z basis)
cf.RightVector              -- +X basis
cf.UpVector                 -- +Y basis

local x, y, z = cf:ToOrientation()       -- Y-X-Z radians
local px, py, pz, rx, ry, rz, ... = cf:GetComponents()
```

### Operators
```lua
cf1 * cf2          -- compose: cf1 then cf2 (applies cf2 in cf1's local space)
cf * Vector3       -- transform a point: world = cf * local
cf:Inverse()       -- inverse transform; cf * cf:Inverse() == identity
cf:ToWorldSpace(local)  -- alt of cf * local
cf:ToObjectSpace(world) -- inverse: world point → cf-local
cf:Lerp(other, t)
cf:VectorToObjectSpace(v) -- rotate a direction (no translation)
cf:VectorToWorldSpace(v)
```

## Patterns

### Move a part forward
```lua
part.CFrame = part.CFrame * CFrame.new(0, 0, -5)  -- 5 studs forward in part's local space
```

### Rotate around its own axis
```lua
part.CFrame = part.CFrame * CFrame.Angles(0, math.rad(90), 0)
```

### Rotate around the world Y axis (about the part's position)
```lua
part.CFrame = CFrame.new(part.Position) * CFrame.Angles(0, math.rad(90), 0) * (part.CFrame - part.Position)
```

### Look at a target
```lua
part.CFrame = CFrame.lookAt(part.Position, target.Position)
```

### Camera following a model
```lua
local cam = workspace.CurrentCamera
cam.CFrame = CFrame.lookAt(model:GetPivot().Position + Vector3.new(0, 10, 20), model:GetPivot().Position)
```

### Spawn a projectile in front of the player
```lua
local hrp = character:FindFirstChild("HumanoidRootPart")
local spawnAt = hrp.CFrame * CFrame.new(0, 0, -3)  -- 3 studs in front
projectile.CFrame = spawnAt
projectile.AssemblyLinearVelocity = spawnAt.LookVector * 100
```

### Rotation interpolation (slerp)
`CFrame:Lerp` does proper rotational interpolation (uses quaternions internally). Don't lerp Orientation values directly — they wrap.

## Pitfalls

- **Using `Orientation` for rotation math.** Orientation is degrees and prone to gimbal/wrap bugs. Always do rotation in CFrame space.
- **Composing in the wrong order.** `A * B` applies `B` in `A`'s local space. To transform around the world axes, factor out position first.
- **Treating LookVector as +Z.** Roblox's forward is **-Z**. `LookVector` is the unit -Z basis.
- **Modifying `cf.Position` directly.** CFrames are immutable. Construct a new one: `cf = CFrame.new(newPos) * (cf - cf.Position)`.
- **Floating-point drift.** Repeated multiplications accumulate error in the rotation matrix. Re-orthonormalize occasionally with `CFrame.lookAt(pos, pos + cf.LookVector, cf.UpVector)` if you do thousands of compositions.
- **`Vector3.Unit` on zero vector** returns NaN. Guard with `if v.Magnitude > 0 then ...`.

## See also
- [parts-and-models.md](parts-and-models.md) — `:PivotTo`
- [camera.md](camera.md) — Camera CFrame
- [raycasting.md](raycasting.md) — directions in queries
