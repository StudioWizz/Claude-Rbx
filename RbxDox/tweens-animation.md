# Tweens & Animation

> Upstream: <https://create.roblox.com/docs/ui/animation>
> Source: `content/en-us/ui/animation.md`

## Purpose
Two systems handle motion in Roblox:
- **TweenService** — interpolates any numeric or numeric-like property of any Instance (Frame `Position`, Part `CFrame`, Light `Brightness`, Sound `Volume`). Use for UI animation, prop animation, camera moves, fades.
- **Animator + AnimationTrack** — plays exported skeletal animations on rigs (characters, NPCs). Use anything driven by a model with `Motor6D` joints.

## TweenService

```lua
local TweenService = game:GetService("TweenService")

local info = TweenInfo.new(
    1.0,                                    -- duration in seconds
    Enum.EasingStyle.Quad,                  -- Linear, Sine, Quad, Cubic, Quart, Quint, Bounce, Elastic, Back, Exponential, Circular
    Enum.EasingDirection.Out,               -- In, Out, InOut
    0,                                      -- repeat count (-1 = infinite)
    false,                                  -- reverses
    0                                       -- delay before start
)

local tween = TweenService:Create(part, info, {
    CFrame = CFrame.new(0, 20, 0),
    Transparency = 0.5,
})
tween:Play()

-- Events
tween.Completed:Connect(function(state)
    -- state: Completed | Cancelled
end)

tween:Cancel()    -- stop, snap to original
tween:Pause()
```

You can tween any property type with linear interpolation: `number`, `Vector2`, `Vector3`, `Color3`, `UDim`, `UDim2`, `CFrame`, `Rect`. Other types you must animate manually with a Heartbeat loop and `:Lerp`.

`TweenService:GetValue(alpha, style, dir)` returns the easing value for a given progress 0–1 — useful for hand-rolled lerps that match TweenInfo curves.

## Easing cheat sheet

- **Linear** — constant rate. Mechanical, no character.
- **Quad / Cubic / Quart / Quint** — accelerating polynomial curves; higher = sharper. `Out` is the default for "natural" UI transitions.
- **Sine** — gentle.
- **Back** — overshoots slightly; springy, polished.
- **Bounce** — multiple bounces at end.
- **Elastic** — oscillates dramatically.

For UI feel: `Quad Out` for slide-in, `Back Out` for pop-in, `Quad InOut` for state transitions.

## AnimationTrack (rigs)

```lua
local anim = Instance.new("Animation")
anim.AnimationId = "rbxassetid://1234567890"

local track = humanoid.Animator:LoadAnimation(anim)
track:Play(fadeTime?, weight?, speed?)
track.Looped = true
track:AdjustSpeed(2.0)
track:AdjustWeight(0.5)
track:Stop(fadeTime?)
track.Priority = Enum.AnimationPriority.Action  -- Idle < Movement < Action < Action4 (later wins overlap)
track.TimePosition = 0
```

Events:
- `track.Stopped`, `track.DidLoop`, `track.KeyframeReached:Connect(function(name) end)`

Animations are authored in the **Animation Editor** (Avatar tab → Animation Editor) on a rig in your place, then published as an asset. Their `AnimationId` is the asset URL.

For NON-character rigs (custom Models with Motor6Ds), put an `AnimationController` Instance in the model with an `Animator` child — same load API.

## Patterns

### Fade UI in
```lua
frame.BackgroundTransparency = 1
local fade = TweenService:Create(frame, TweenInfo.new(0.3), {BackgroundTransparency = 0.2})
fade:Play()
```

### Slide menu in from the right
```lua
menu.Position = UDim2.new(1, 0, 0.5, 0)             -- offscreen right
TweenService:Create(menu,
    TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
    {Position = UDim2.new(0.7, 0, 0.5, 0)}
):Play()
```

### Pulse effect
```lua
TweenService:Create(button,
    TweenInfo.new(0.4, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true),
    {Size = button.Size + UDim2.fromOffset(10, 10)}
):Play()
```

### Cinematic camera move
```lua
TweenService:Create(workspace.CurrentCamera,
    TweenInfo.new(2, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut),
    {CFrame = endCFrame}
):Play()
```

### Crossfade animations
```lua
walk:Stop(0.3)        -- fade out
run:Play(0.3)         -- fade in over 0.3s
```

## Pitfalls

- **Tweening Anchored=false parts.** Physics fights the tween. Anchor the part for the duration, or use a constraint instead.
- **Tweening Position on a UIListLayout child.** Layout overrides you instantly. Tween LayoutOrder or wrap in a parent frame.
- **Forgetting `:Cancel()`.** Stale tweens leak in long-running scripts.
- **Re-tweening every frame.** Each `:Create` is a new object. Cache the tween if it's recurring.
- **Animations loaded on a not-yet-replicated Animator** error. Wait: `humanoid:WaitForChild("Animator")`.
- **Wrong AnimationPriority.** A Movement-priority anim won't override the Action-priority idle if you have it backwards. Set Priority to match intent.
- **Server-side `LoadAnimation`** still works but the modern path is to load on the client (or both, at appropriate priorities) for snappy response.

## See also
- [characters.md](characters.md) — Animator/Humanoid
- [ui-system.md](ui-system.md) — what UDim2 properties to tween
- [task-scheduling.md](task-scheduling.md) — manual lerps with Heartbeat
- [camera.md](camera.md), [lighting.md](lighting.md) — common tween targets (cinematic moves, color-grade transitions)
- [interactions.md](interactions.md), [vfx.md](vfx.md), [sound.md](sound.md) — tween Volume, Beam properties, door swings on Triggered
