# Lighting & Atmosphere

> Upstream: <https://create.roblox.com/docs/environment/lighting> · <https://create.roblox.com/docs/reference/engine/classes/Lighting>
> Source: distilled from the Lighting service and post-effect class references

## Purpose
The `Lighting` service controls global illumination, time of day, fog, atmosphere, and screen-space post-effects. It's mostly a **designer / builder concern** — values you tune in Studio and rarely touch from code — but it's worth knowing what's there and which properties are runtime-friendly when you do need them (e.g., dynamic time-of-day, "lights out" effect, color-grade swap on entering a zone).

## Lighting service — key properties

```lua
local Lighting = game:GetService("Lighting")

Lighting.ClockTime         -- 0..24 (real-time hour); also TimeOfDay = "HH:MM:SS"
Lighting.GeographicLatitude -- 0 = equator; affects sun angle
Lighting.Ambient           -- Color3, indirect light tint on shadowed surfaces
Lighting.OutdoorAmbient    -- Color3, ambient for outdoor (skybox-lit) areas
Lighting.Brightness        -- 0..10, sun/moon intensity
Lighting.ColorShift_Top    -- Color3, tints the top of objects (sky reflection)
Lighting.ColorShift_Bottom -- Color3, tints the bottom (ground reflection)
Lighting.ShadowSoftness    -- 0..1
Lighting.EnvironmentDiffuseScale, EnvironmentSpecularScale
Lighting.GlobalShadows     -- bool (master shadow toggle)
Lighting.Technology        -- Voxel | ShadowMap | Future | Legacy (lighting engine)
Lighting.ExposureCompensation
```

Set `Technology = Future` for the modern PBR lighting engine — needed for full SurfaceAppearance fidelity (see [parts-and-models.md](parts-and-models.md)).

## Children of Lighting

These Instances live as children of `Lighting` and apply globally:

- **`Atmosphere`** — fog/haze with density, offset, color, decay. Modern replacement for the old FogStart/FogEnd properties on Lighting itself.
- **`Sky`** — custom skybox (six face textures or a SkyboxBk/Up/Dn/Lf/Rt/Ft set).
- **`Clouds`** — procedural cloud layer (parented under `Terrain`, but tuned alongside Lighting).
- **Post-effects** (parented under Lighting):
  - `BloomEffect` — glow on bright pixels.
  - `BlurEffect` — full-screen blur (use sparingly; expensive on mobile).
  - `ColorCorrectionEffect` — Brightness/Contrast/Saturation/TintColor — the cheapest way to globally re-grade.
  - `DepthOfFieldEffect` — focus blur (FocusDistance, InFocusRadius, NearIntensity/FarIntensity).
  - `SunRaysEffect` — god-ray streaks from the sun.

## Patterns

### Day/night cycle
```lua
local Lighting = game:GetService("Lighting")
local RunService = game:GetService("RunService")

-- 5-minute full day
local SECONDS_PER_DAY = 300

RunService.Heartbeat:Connect(function(dt)
    Lighting.ClockTime = (Lighting.ClockTime + dt * 24 / SECONDS_PER_DAY) % 24
end)
```

### Tween between two color grades (e.g., entering a cave)
```lua
local TweenService = game:GetService("TweenService")
local cc = Lighting:FindFirstChildOfClass("ColorCorrectionEffect")

TweenService:Create(cc, TweenInfo.new(1.5), {
    Brightness = -0.2,
    Saturation = -0.5,
    TintColor = Color3.fromRGB(120, 140, 180),
}):Play()
```

### "Lights out" flash
```lua
local cc = Lighting:FindFirstChildOfClass("ColorCorrectionEffect")
local tween = TweenService:Create(cc, TweenInfo.new(0.1), {Brightness = -1})
tween:Play()
tween.Completed:Wait()
TweenService:Create(cc, TweenInfo.new(0.5), {Brightness = 0}):Play()
```

### Per-player lighting (rare but useful)
Lighting changes on the server replicate to all clients. For per-player effects (e.g., a player drinks a potion that desaturates only their view), use a **client-side `ColorCorrectionEffect`** parented under their PlayerGui — the engine respects post-effects on either side.

```lua
-- LocalScript
local cc = Instance.new("ColorCorrectionEffect")
cc.Saturation = -0.8
cc.Parent = game:GetService("Lighting")    -- still under Lighting; it's just a client-only Instance now
task.delay(10, function() cc:Destroy() end)
```

## Pitfalls

- **`Technology = Voxel`** (legacy default in some old places) doesn't render PBR materials, normal maps, or SurfaceAppearance correctly. Use `Future` for modern visuals.
- **Heavy post-effects on mobile** — Bloom, DOF, SunRays cost real frame time on lower-end devices. Test perf with `MicroProfiler` (see [performance.md](performance.md)).
- **Tweening `Lighting.Ambient` directly** is subtle and easy to miss. ColorCorrectionEffect.TintColor is more obvious and easier to art-direct.
- **`ClockTime` snap**: setting from 23.9 to 0.1 is a 24-hour skip in either direction. For continuous transitions, lerp manually.
- **Fog feel wrong** — the legacy `Lighting.FogStart`/`FogEnd` still works but the modern `Atmosphere` Instance gives much better results. Add an Atmosphere child if you don't have one.
- **No "per-zone" Lighting natively.** Use overlap detection ([raycasting.md](raycasting.md)) plus client-side tweens to fake it. Or use `Atmosphere` + region-based Color3 lerps.
- **Lighting changes from a `LocalScript`** affect only that client — fine for cinematics, surprising if you forget.

## See also
- [time-and-weather.md](time-and-weather.md) — game-time-driven day/night cycle, weather state machine, seasonal cycles (drives Lighting from above)
- [areas-and-biomes.md](areas-and-biomes.md) — per-area lighting presets swap on region transition
- [parts-and-models.md](parts-and-models.md) — materials and SurfaceAppearance interact with Lighting.Technology
- [tweens-animation.md](tweens-animation.md) — TweenService for smooth lighting transitions
- [performance.md](performance.md) — post-effect costs
- [services.md](services.md) — Lighting overview
