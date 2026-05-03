# Physics Constraints

> Upstream: <https://create.roblox.com/docs/physics/constraints>
> Source: `content/en-us/physics/mechanical-constraints.md`, `mover-constraints.md`, plus the per-constraint pages

## Purpose
Constraints connect parts in physically simulated ways — hinges, springs, ropes, motors. They're the modern replacement for old `Weld`/`Motor` instances and they expose proper physics simulation. Use constraints any time you want non-rigid relationships between parts (doors, vehicles, ragdolls, projectiles).

Two broad categories:
- **Mechanical constraints** define geometric relationships (hinge, weld, ball-socket).
- **Mover constraints** apply forces/velocities (LinearVelocity, AlignPosition, AngularVelocity, Torque).

All constraints attach to two `Attachment` Instances (one in each connected part). The Attachment defines the local pivot/orientation.

## Setting up an attachment-based constraint
```lua
-- Two attachments, one in each part, oriented as needed
local a0 = Instance.new("Attachment", partA)
a0.Position = Vector3.new(0, 1, 0)

local a1 = Instance.new("Attachment", partB)
a1.Position = Vector3.new(0, -1, 0)

local hinge = Instance.new("HingeConstraint")
hinge.Attachment0 = a0
hinge.Attachment1 = a1
hinge.Parent = partA
```

## Mechanical constraints (geometry)

| Constraint | Restricts |
|---|---|
| `WeldConstraint` | All 6 DOF — rigid bond. Fastest for static joins. |
| `RigidConstraint` | All 6 DOF, attachment-based (preferred over `Weld`). |
| `HingeConstraint` | 5 DOF; rotates freely on Z axis. Add `MotorMaxTorque`/`AngularVelocity` for powered hinge. |
| `BallSocketConstraint` | Translation locked, rotation free (or limited via `LimitsEnabled`). Ragdoll joints. |
| `PrismaticConstraint` | Sliding only along one axis (piston). |
| `CylindricalConstraint` | Slide + rotate on one axis. |
| `UniversalConstraint` | Like a U-joint. |
| `RopeConstraint` | Max distance; floppy. `Length` is the cap. |
| `RodConstraint` | Fixed distance. |
| `SpringConstraint` | Damped spring; `Stiffness`, `Damping`, `FreeLength`. |
| `TorsionSpringConstraint` | Rotational spring around an axis. |
| `PlaneConstraint` | Constrains motion to a plane. |
| `NoCollisionConstraint` | Disables collisions between two specific parts (no mass, no force — just collision filter). |

## Mover constraints (force/velocity)

| Constraint | Effect |
|---|---|
| `AlignPosition` | Pulls Attachment0 toward Attachment1 (or a target world position with `Mode = OneAttachment`). Tunable `MaxForce`, `Responsiveness`. |
| `AlignOrientation` | Same but for orientation. |
| `LinearVelocity` | Drives a target linear velocity. `MaxForce` cap. |
| `AngularVelocity` | Drives a target angular velocity. |
| `VectorForce` | Applies a constant force in attachment-local or world coords. |
| `Torque` | Applies a constant torque. |
| `LineForce` | Force along the line between two attachments. |

For mover constraints, `RelativeTo` (Attachment0 / Attachment1 / World) controls reference frame. `MaxForce`/`MaxTorque` cap how hard the engine pushes — set high for snappy, low for soft.

## Patterns

### Door on a hinge
```lua
local hinge = Instance.new("HingeConstraint")
hinge.Attachment0 = doorAttachment
hinge.Attachment1 = frameAttachment
hinge.LimitsEnabled = true
hinge.LowerAngle = -90
hinge.UpperAngle = 0
hinge.Parent = door
```

### Powered hinge (motor)
```lua
hinge.ActuatorType = Enum.ActuatorType.Motor
hinge.MotorMaxTorque = 10000
hinge.AngularVelocity = math.rad(180)  -- spin at 180°/sec
```

### Hover / float
Use `AlignPosition` with a target attachment in the air:
```lua
local align = Instance.new("AlignPosition")
align.Attachment0 = bodyAttachment
align.Mode = Enum.PositionAlignmentMode.OneAttachment
align.Position = Vector3.new(0, 10, 0)
align.MaxForce = 1e6
align.Responsiveness = 50
align.Parent = body
```

### Disable collision between two parts
```lua
local nc = Instance.new("NoCollisionConstraint")
nc.Part0 = a
nc.Part1 = b
nc.Parent = a
```
Faster than collision groups for one-off pair exclusions.

### Replace a Weld with WeldConstraint
```lua
local w = Instance.new("WeldConstraint")
w.Part0 = a
w.Part1 = b
w.Parent = a
-- That's it; relative offset is baked from current CFrames.
```

## Physical properties

Each part has implicit `PhysicalProperties` from its material; override with `CustomPhysicalProperties`:
```lua
part.CustomPhysicalProperties = PhysicalProperties.new(
    0.7,    -- density (g/cm³ish)
    0.3,    -- friction
    0.5,    -- elasticity (bounciness)
    1,      -- frictionWeight
    1       -- elasticityWeight
)
```

`Massless = true` makes a part contribute no mass (good for visual-only attachments to a vehicle frame).

## Network ownership

The server normally hands physics simulation of an unanchored assembly to the closest player's client. To force server simulation:
```lua
part:SetNetworkOwner(nil)
```
Use this for projectiles, hazards, NPC bodies — anything where client-side physics would let the owner cheat. See [security.md](security.md).

## Pitfalls

- **`Anchored` parts ignore constraints.** Constraints only act between two unanchored, simulated parts.
- **`Massless`** without an unmassless ally produces zero net force.
- **`MaxForce` too low** → constraint visibly fails to track. Crank it up for snappy controls.
- **`Responsiveness` too low** → mushy lag. Default 5 is gentle; 50+ for snappy.
- **Old `Weld` / `Motor` / `Motor6D` joints.** Motor6D is still used for character animations (joints between body parts driven by Animator). For everything else, prefer Constraints.
- **Constraints between parts in different assemblies cause unexpected relative motion** — make sure the bodies you're connecting are actually independent.

## See also
- [parts-and-models.md](parts-and-models.md) — BasePart properties
- [collisions.md](collisions.md) — CanCollide, NoCollisionConstraint, CollisionGroups
- [characters.md](characters.md) — Motor6D in characters
- [vehicles.md](vehicles.md) — applied use of HingeConstraint motors (wheels) and SpringConstraint suspension
