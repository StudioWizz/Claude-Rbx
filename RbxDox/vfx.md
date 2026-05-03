# VFX (Particles, Beams, Trails)

> Upstream: <https://create.roblox.com/docs/effects> · ParticleEmitter / Beam / Trail API references
> Source: distilled from the engine VFX class references

## Purpose
Roblox's built-in visual-effect Instances cover most needs without writing a particle system from scratch. **`ParticleEmitter`** is the workhorse for fire, smoke, sparks, hits, ambience. **`Beam`** draws a textured strip between two points (lasers, lightning, ropes, magic). **`Trail`** draws a streak behind moving objects. **`Smoke`/`Fire`/`Sparkles`** are legacy one-property emitters kept for backward compatibility — usable for quick effects.

All of these are **visual only** — they don't collide, can't be raycast against, and don't fire events. They live as children of a `BasePart` (or, for Trail/Beam, between two `Attachment`s).

## ParticleEmitter

Parent under a `BasePart`; emits from the part's volume by default.

### Key properties
```lua
emitter.Texture            -- "rbxassetid://..." — required
emitter.Rate               -- particles per second (continuous)
emitter.Lifetime           -- NumberRange.new(min, max) — seconds each particle lives
emitter.Speed              -- NumberRange — initial speed
emitter.SpreadAngle        -- Vector2 — cone angle in degrees (X, Y axes)
emitter.Rotation           -- NumberRange — initial rotation degrees
emitter.RotSpeed           -- NumberRange — degrees/sec
emitter.Drag               -- 0 = no air resistance; positive slows particles
emitter.Acceleration       -- Vector3 — applied each frame (e.g. Vector3.new(0,-50,0) for gravity)
emitter.EmissionDirection  -- Enum.NormalId — Top/Front/Back/etc. of the parent part

-- Sequenced over each particle's lifetime (0..1):
emitter.Color              -- ColorSequence
emitter.Size               -- NumberSequence
emitter.Transparency       -- NumberSequence

emitter.LightEmission      -- 0..1 — how much they glow (additive blending)
emitter.LightInfluence     -- 0..1 — how much world light affects them
emitter.ZOffset            -- pull toward camera; positive = in front of overlapping geometry
emitter.Orientation        -- FacingCamera | FacingCameraWorldUp | VelocityParallel | VelocityPerpendicular
emitter.Enabled            -- bool; turn off to stop continuous emission
emitter.LockedToPart       -- if true, particles move with the part; if false, they're released into world space
```

### Methods
```lua
emitter:Emit(particleCount)   -- one-shot burst
emitter:Clear()               -- remove all currently-alive particles immediately
```

### Patterns

**One-shot hit burst** (the most common pattern):
```lua
-- Pre-author an emitter inside a template Part, set Enabled = false (so it doesn't continuously emit).
-- At runtime: clone, position, :Emit(n), Debris-clean.

local Debris = game:GetService("Debris")

local function spawnHit(position: Vector3)
    local p = Instance.new("Part")
    p.Anchored = true
    p.CanCollide = false
    p.CanQuery = false
    p.Transparency = 1
    p.Size = Vector3.one
    p.Position = position
    p.Parent = workspace.Effects     -- a Folder; see workspace-hierarchy.md

    local burst = template.HitParticles:Clone()
    burst.Parent = p
    burst:Emit(30)

    Debris:AddItem(p, 2)
end
```

**Continuous trail (low rate):**
```lua
local fire = Instance.new("ParticleEmitter")
fire.Texture = "rbxassetid://7233853628"   -- example fire wisp
fire.Rate = 20
fire.Lifetime = NumberRange.new(0.5, 1.5)
fire.Speed = NumberRange.new(2, 4)
fire.Color = ColorSequence.new(Color3.fromRGB(255, 180, 50), Color3.fromRGB(255, 50, 0))
fire.Size = NumberSequence.new({
    NumberSequenceKeypoint.new(0, 0.5),
    NumberSequenceKeypoint.new(1, 1.5),
})
fire.Transparency = NumberSequence.new({
    NumberSequenceKeypoint.new(0, 0),
    NumberSequenceKeypoint.new(1, 1),
})
fire.Acceleration = Vector3.new(0, 4, 0)
fire.LightEmission = 0.6
fire.Parent = torchHeadPart
```

**Toggle on/off:** flip `Enabled`. Particles already emitted continue their lifetime; new ones stop spawning.

## Beam

Two `Attachment`s on (possibly different) parts. The Beam renders a textured strip between them every frame.

```lua
beam.Attachment0 = a
beam.Attachment1 = b
beam.Texture = "rbxassetid://..."
beam.Color = ColorSequence.new(Color3.new(1,1,1))
beam.Width0 = 0.5
beam.Width1 = 0.5
beam.CurveSize0 = 5         -- pull the line away from straight (Bezier handles)
beam.CurveSize1 = -5
beam.Segments = 10          -- subdivisions; higher = smoother curve
beam.Transparency = NumberSequence.new(0)
beam.FaceCamera = true      -- billboard the strip toward the camera
beam.LightEmission = 0
beam.TextureSpeed = 1       -- scrolls the texture along the beam (laser shimmer)
beam.TextureMode = Enum.TextureMode.Stretch | Wrap | Static
```

### Patterns

**Laser between two parts:**
```lua
local a = Instance.new("Attachment", emitterPart)
local b = Instance.new("Attachment", targetPart)

local beam = Instance.new("Beam")
beam.Attachment0 = a
beam.Attachment1 = b
beam.Width0 = 0.2; beam.Width1 = 0.2
beam.Color = ColorSequence.new(Color3.new(1, 0, 0))
beam.LightEmission = 1
beam.Texture = "rbxassetid://446111271"
beam.TextureSpeed = 2
beam.Parent = emitterPart
```

For a **lightning bolt**, increase `Segments`, set CurveSize0/1 to random values, and randomize the attachment positions slightly each frame.

## Trail

Two `Attachment`s on the **same part**. The trail renders a streak between them as the part moves.

```lua
trail.Attachment0 = topAttachment
trail.Attachment1 = bottomAttachment
trail.Texture = "rbxassetid://..."
trail.Color = ColorSequence.new(Color3.new(1, 1, 1))
trail.Lifetime = 0.5            -- seconds the streak persists
trail.Transparency = NumberSequence.new({
    NumberSequenceKeypoint.new(0, 0),
    NumberSequenceKeypoint.new(1, 1),
})
trail.WidthScale = NumberSequence.new(1)
trail.MinLength = 0.1            -- discard segments shorter than this (hides jitter)
trail.MaxLength = 0              -- 0 = unlimited
trail.FaceCamera = false
trail.LightEmission = 0
trail.LightInfluence = 1
```

Use trails on projectiles, swords during a swing, vehicles, dashing characters.

## Smoke / Fire / Sparkles (legacy)

One-property "just slap it on a part" emitters. Less control than ParticleEmitter but zero setup.

```lua
local fire = Instance.new("Fire", part)
fire.Size = 5
fire.Heat = 9
fire.Color = Color3.new(1, 0.5, 0)
fire.SecondaryColor = Color3.new(0.5, 0, 0)
```

For new effects, prefer ParticleEmitter — it's more flexible and consistent with Beam/Trail.

## Patterns (cross-cutting)

### Effects folder
Keep a top-level `workspace.Effects` Folder for runtime-created effect parts. Cleanup with `Debris:AddItem`. See [workspace-hierarchy.md](workspace-hierarchy.md).

### Pair with sound
```lua
spawnHit(pos)
local s = Instance.new("Sound")
s.SoundId = "rbxassetid://..."
s.Parent = effectPart   -- 3D positional
s.PlayOnRemove = true
Debris:AddItem(s, 0.1)  -- destroy → plays once → cleaned up
```
See [sound.md](sound.md) for the PlayOnRemove pattern.

### Pre-authored vs code-built
For complex emitters with sequences, **author them in Studio inside a template Part**, then `:Clone()` at runtime. Building a NumberSequence/ColorSequence in code is verbose and hard to tweak.

## Performance

Particles are cheap individually but death by thousand cuts is real:
- **`Rate × max(Lifetime)` = particles on screen.** A `Rate = 200`, `Lifetime = 5` emitter holds 1,000 alive at any moment. Scale down for ambient effects.
- **Avoid hundreds of simultaneous emitters.** Pool effect parts; reuse rather than spawning new ones every frame.
- **`Beam.Segments` cost is per beam per frame.** 10 is the default; reduce for many simultaneous beams.
- **`Trail.MinLength`** filters out tiny segments — use it on fast-moving objects to avoid spam.
- **Stop, don't destroy, when toggling.** Setting `Enabled = false` is cheaper than destroying and re-creating an emitter.

See [performance.md](performance.md) for general optimization principles.

## Pitfalls

- **No `Texture` set** → particles render as the default white square, ugly. Texture is effectively required.
- **Forgot `:Emit(n)` because `Enabled = false`** — Emit works regardless of Enabled. If you also leave Enabled = true, both fire.
- **Particles disappearing behind walls** → set `ZOffset` slightly positive, or use `LightEmission = 1` for additive blending that shows through smoke.
- **Beam invisible** → check both Attachments are in `Workspace`, parts are in line of sight (Beam doesn't care, but if your Attachments are wrong they may be at origin), Width > 0, Texture set.
- **Trail streaks weirdly when the part teleports** → trails interpolate between frames. After a `:PivotTo` jump, briefly disable the Trail then re-enable (or `:Clear()` it on a ParticleEmitter sibling).
- **Attachments oriented wrong on Beam** → Beam orientation depends on Attachment LookVector. If the strip looks twisted, rotate the Attachment.
- **`LockedToPart = true` with a moving part** — particles drag along, which often isn't what "smoke from a moving torch" wants. Default `false` releases them into world space.
- **Server-side `:Emit(n)` for one client** → all clients see the burst. To play for one player only, fire a RemoteEvent and have the client emit.
- **Effects parts not destroyed** → leak Workspace children. Always `Debris:AddItem` or explicit `:Destroy()`.
- **Audio missing context** — particles without a paired sound feel cheap. Almost every hit/explosion benefits from a Sound, even a quiet one. See [sound.md](sound.md).

## See also
- [parts-and-models.md](parts-and-models.md) — anchored/CanCollide/CanQuery for effect parts
- [workspace-hierarchy.md](workspace-hierarchy.md) — Effects folder pattern; build-then-parent
- [sound.md](sound.md) — pair effects with audio
- [tweens-animation.md](tweens-animation.md) — animate Beam/Trail properties for richer effects
- [performance.md](performance.md) — particle counts, pooling
- [services.md](services.md) — Debris service for auto-cleanup
