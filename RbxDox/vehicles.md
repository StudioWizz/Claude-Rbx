# Vehicles & Seats

> Upstream: <https://create.roblox.com/docs/reference/engine/classes/VehicleSeat> · <https://create.roblox.com/docs/reference/engine/classes/Seat>
> Source: distilled from the Seat / VehicleSeat / HingeConstraint API references and standard Roblox vehicle patterns

## Purpose
A Roblox vehicle is just a `Model` of welded `Part`s with a **`VehicleSeat`** that routes player steering input, plus motorized **`HingeConstraint`**s for wheels (or `LinearVelocity`/`AlignPosition` for hovers, planes, boats). The engine handles "player sits in seat → engine takes inputs → wheel motors apply torque" if you wire it the standard way. There's no `Vehicle` class — vehicles are an *assembly pattern*, not a single Instance.

For non-vehicle sitting (chairs, couches, mounts a player rides without controlling), use the simpler **`Seat`** class.

## Seat (passive sitting)

```lua
-- Properties
seat.Disabled              -- bool; true = no one can sit
seat.Occupant              -- Humanoid currently sitting (or nil)

-- Events
seat.ChildAdded            -- a Weld is created/destroyed here as the player sits/stands
```

When a player walks into a Seat, the engine welds their HumanoidRootPart to the Seat via a `SeatWeld` child. Setting `humanoid.Sit = false` ejects them. Setting `seat.Disabled = true` after they sit keeps them seated; toggle before they touch.

```lua
seat.ChildAdded:Connect(function(child)
    if child:IsA("Weld") and child.Name == "SeatWeld" then
        local hum = child.Part1 and child.Part1.Parent and child.Part1.Parent:FindFirstChildOfClass("Humanoid")
        if hum then onSit(hum) end
    end
end)
```

A nicer, more reliable signal:
```lua
seat:GetPropertyChangedSignal("Occupant"):Connect(function()
    if seat.Occupant then
        onSit(seat.Occupant)
    else
        onStand()
    end
end)
```

## VehicleSeat (steering input routing)

A `VehicleSeat` is a `Seat` that automatically reads the seated player's WASD / left-stick input and exposes it as properties. You read those and apply them to your motors / forces.

```lua
-- Properties (Roblox sets these from the driver's input)
vehicleSeat.Throttle       -- -1 (S/back) | 0 (none) | 1 (W/forward)
vehicleSeat.Steer          -- -1 (A/left) | 0 (none) | 1 (D/right)
vehicleSeat.ThrottleFloat  -- analog version, gamepad triggers/stick
vehicleSeat.SteerFloat     -- analog version

-- Settings (you configure)
vehicleSeat.MaxSpeed       -- units/second (informational; doesn't enforce)
vehicleSeat.Torque         -- legacy property; modern code uses HingeConstraint motors directly
vehicleSeat.TurnSpeed      -- legacy
vehicleSeat.HeadsUpDisplay -- bool; show speedometer

-- Same as Seat:
vehicleSeat.Occupant
vehicleSeat.Disabled
```

The legacy `Torque`/`TurnSpeed` properties only work for old-style "wheels = parts with `Motor` joints" vehicles (Roblox auto-rigs them when you parent a `VehicleSeat` correctly). **Modern vehicles ignore those and drive HingeConstraint motors explicitly** based on `Throttle`/`Steer`.

## Modern car pattern — motorized HingeConstraints

```
Car (Model, PrimaryPart = Body)
├── Body (BasePart, large, low-poly)
├── VehicleSeat (welded to Body)
├── Wheels (Folder)
│   ├── FrontLeft (BasePart, ball/cylinder)
│   │   ├── Attachment (in Wheel, axis aimed along the rotation axis)
│   │   └── HingeConstraint (Attachment0 = body attachment, Attachment1 = wheel attachment, ActuatorType = Motor)
│   ├── FrontRight (...)
│   ├── RearLeft (...)
│   └── RearRight (...)
└── Server Script (drives the motors from VehicleSeat input)
```

```lua
-- Server Script under the Car model
local car = script.Parent
local seat = car:WaitForChild("VehicleSeat") :: VehicleSeat

local SPEED = 50            -- target angular velocity at full throttle (rad/s)
local TURN_ANGLE = 0.3      -- max steering deflection (rad)

local rearMotors = {
    car.Wheels.RearLeft.HingeConstraint,
    car.Wheels.RearRight.HingeConstraint,
}
local frontHinges = {
    car.Wheels.FrontLeft.HingeConstraint,
    car.Wheels.FrontRight.HingeConstraint,
}

-- Drive: rear-wheel motors get throttle * SPEED
local function update()
    local throttle = seat.ThrottleFloat
    local steer = seat.SteerFloat
    for _, m in rearMotors do
        m.AngularVelocity = -throttle * SPEED
        m.MotorMaxTorque = if seat.Occupant then 50000 else 0
    end
    for _, h in frontHinges do
        h.TargetAngle = steer * math.deg(TURN_ANGLE)
        h.AngularSpeed = 5
    end
end

-- Update every step while someone's driving
game:GetService("RunService").Heartbeat:Connect(update)

-- Hand network ownership to the driver for snappy feel; revoke when empty
seat:GetPropertyChangedSignal("Occupant"):Connect(function()
    local driver = seat.Occupant and game:GetService("Players"):GetPlayerFromCharacter(seat.Occupant.Parent)
    for _, p in car:GetDescendants() do
        if p:IsA("BasePart") then
            p:SetNetworkOwner(driver)   -- nil = server
        end
    end
end)
```

Front wheels use **steering** hinges (`ActuatorType = Servo`, `TargetAngle` set per frame). Rear wheels use **drive** hinges (`ActuatorType = Motor`, `AngularVelocity` set per frame). Rear-wheel-drive is the simplest stable layout; AWD doubles the motors and complicates handling.

## Suspension

Add `SpringConstraint`s between the body and the wheel-housing to absorb bumps:
```lua
local spring = Instance.new("SpringConstraint")
spring.Attachment0 = bodyAttachment
spring.Attachment1 = wheelHousingAttachment
spring.FreeLength = 1.5
spring.Stiffness = 8000
spring.Damping = 200
spring.LimitsEnabled = true
spring.MinLength = 1
spring.MaxLength = 2
spring.Parent = car
```

Tune `Stiffness` for ride feel: low = floaty, high = stiff. Damping prevents oscillation. See [physics-constraints.md](physics-constraints.md).

## Network ownership — the critical perf decision

By default, an unanchored Model near a player gets that player's client as its physics owner. For a vehicle:

- **Driver-owned** (most cars): the driver simulates physics, vehicle feels snappy for them, other clients see slightly-smoothed replicated state. Set ownership to the driver when they sit; nil out when they leave.
- **Server-owned** (PvP cars where exploits matter): the server simulates everything, all clients see input lag, but no driver can teleport their car. Use `:SetNetworkOwner(nil)` on every part.

The pattern in the example above gives ownership to the driver — fine for most games. For competitive/PvP vehicles, hold ownership server-side and accept the latency. See [security.md](security.md).

## Other vehicle types

### Hovercraft / floating
Use **`AlignPosition`** in `OneAttachment` mode to hold a target Y position, plus **`LinearVelocity`** for forward thrust and **`AngularVelocity`** for turning.
```lua
local hover = Instance.new("AlignPosition")
hover.Mode = Enum.PositionAlignmentMode.OneAttachment
hover.Attachment0 = bodyAttachment
hover.Position = bodyAttachment.WorldPosition + Vector3.new(0, 5, 0)  -- target hover height
hover.MaxForce = 1e6
hover.Responsiveness = 50
hover.Parent = body

-- Steer/thrust based on Throttle/Steer in a Heartbeat loop, like the car example
```

### Plane
- `LinearVelocity` along Look for thrust
- `Torque` or `AngularVelocity` for pitch/yaw/roll
- `BodyGyro`-style stabilization via `AlignOrientation` (pulls toward level)
- Read `Throttle` for thrust, `Steer` for yaw, mouse for pitch (mouse on client → RemoteEvent → server)

### Boat
- `LinearVelocity` for thrust + steering torque, but parent the assembly such that water buoyancy works (terrain water provides natural floatation — see `Workspace.Terrain`)

## Patterns

### Auto-eject when leaving a seat
```lua
seat:GetPropertyChangedSignal("Occupant"):Connect(function()
    if not seat.Occupant then
        -- car cleanup: stop motors, return ownership to server
        for _, m in motors do m.MotorMaxTorque = 0 end
        for _, p in car:GetDescendants() do
            if p:IsA("BasePart") then p:SetNetworkOwner(nil) end
        end
    end
end)
```

### One-key honk / lights
Listen for `seat.Occupant` and bind via ContextActionService on the client when it changes:
```lua
-- LocalScript under StarterPlayerScripts
local seat = workspace:WaitForChild("Car"):WaitForChild("VehicleSeat")
local CAS = game:GetService("ContextActionService")

seat:GetPropertyChangedSignal("Occupant"):Connect(function()
    local me = game:GetService("Players").LocalPlayer.Character
    if seat.Occupant and seat.Occupant.Parent == me then
        CAS:BindAction("Honk", function(_, state)
            if state == Enum.UserInputState.Begin then honkRemote:FireServer() end
        end, false, Enum.KeyCode.H, Enum.KeyCode.ButtonR1)
    else
        CAS:UnbindAction("Honk")
    end
end)
```

### Damaging vehicle (taking hits, exploding)
Use Attributes on the Body for HP; tag the assembly with `CollectionService`. Damage RemoteEvents validated server-side, just like player damage. See [security.md](security.md), [tags.md](tags.md), [attributes.md](attributes.md).

## Pitfalls

- **`VehicleSeat.Throttle`/`Steer` are integer-3-state by default** (-1/0/1). For analog gamepad input, read `ThrottleFloat`/`SteerFloat`.
- **Mass mismatch.** Vehicles with a heavy body and tiny wheels handle terribly. Match wheel mass to body, or use `Massless = true` on visual-only parts.
- **Wheel HingeConstraint axis wrong.** The Attachment's orientation defines the rotation axis. If wheels spin sideways or oddly, rotate the Attachment.
- **Forgetting to give network ownership to the driver.** The vehicle will feel laggy for them as the server simulates remotely. Always set ownership on Occupant change.
- **Server-authoritative vehicles AND `SetNetworkOwner(driver)` mixed up.** Pick one model. For PvP, server-owned. For racing, driver-owned.
- **Spawning a vehicle on top of the player's head.** Models clip into the world if you don't raycast for ground first.
- **Old `Motor` joint vehicles** still work but are deprecated. New code uses `WeldConstraint` + `HingeConstraint` exclusively.
- **Players exiting via Backspace at speed** ragdoll across the world. If undesired, set `seat.Disabled = true` then teleport them out manually.
- **`seat.Occupant` references the Humanoid, not the Player.** Use `Players:GetPlayerFromCharacter(occupant.Parent)` to get the Player.
- **Chained vehicles (trailer, towed)** — connect with a `RopeConstraint` or `BallSocketConstraint`, not Welds, for natural pivoting.

## See also
- [physics-constraints.md](physics-constraints.md) — HingeConstraint motor mode, SpringConstraint, AlignPosition, LinearVelocity
- [collisions.md](collisions.md) — wheel collision groups, network ownership
- [characters.md](characters.md) — Humanoid Sit/Jump for players in seats
- [security.md](security.md) — server vs driver ownership trade-off
- [input-handling.md](input-handling.md) — ContextActionService for in-vehicle actions (honk, lights, weapons)
- [world-mechanics.md](world-mechanics.md) — SurfaceVelocity (alternative to wheels for treadmills/conveyors)
