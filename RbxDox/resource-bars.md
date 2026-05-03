# Resource Bars (Health, Stamina, Mana, Shield, XP, Ammo)

> Source: distilled from resource-pool patterns across RPG / FPS / fighting Roblox games

## Purpose
Most games have several **fillable resources** the player tracks: HP, stamina, mana, shield, XP, ammo, durability, oxygen, hunger. They share a common pattern — a current value, a max, regen rules, depletion triggers — and they're displayed as **fill bars** in the HUD. This card unifies the resource-pool model (data shape, regen discipline, pool interactions) and the bar-UI primitive (segmented vs gradient, color thresholds, smooth fill animation). Mentions of stamina/mana in `combat.md` and the XP bar in `progression.md` are specific applications of this general pattern.

## The unified resource pattern

```lua
--!strict
export type Resource = {
    current: number,
    max: number,
    regenPerSec: number,           -- 0 = no regen
    regenDelaySec: number,         -- seconds after last drain before regen kicks in
    lastDrainAt: number,           -- os.clock()
    blockingFlags: {[string]: boolean}?,   -- e.g., {sprinting = true} blocks stamina regen
}

-- Per-character / per-player state
type CombatState = {
    health: Resource,        -- mirrors Humanoid.Health/MaxHealth (engine-managed)
    stamina: Resource,
    mana: Resource,
    shield: Resource?,       -- optional secondary HP pool
    -- XP and ammo are simpler — see specialized sections below
}
```

Health is *also* a resource but it's already managed by `Humanoid.Health`/`MaxHealth`. Treat it as a Resource conceptually (so the bar UI is uniform) but read/write via the Humanoid's properties.

## Regen rules

The discipline:
1. **Drain** sets `lastDrainAt = os.clock()` and reduces `current`.
2. **Regen** runs every Heartbeat. It only fires if:
   - `current < max` (not already full)
   - `os.clock() - lastDrainAt >= regenDelaySec` (delay elapsed)
   - No `blockingFlags` are set (sprinting blocks stamina regen, casting blocks mana regen, etc.)

```lua
function Resource.Drain(r: Resource, amount: number)
    r.current = math.max(0, r.current - amount)
    r.lastDrainAt = os.clock()
end

function Resource.Restore(r: Resource, amount: number)
    r.current = math.min(r.max, r.current + amount)
end

function Resource.Tick(r: Resource, dt: number)
    if r.current >= r.max then return end
    if os.clock() - r.lastDrainAt < r.regenDelaySec then return end
    if r.blockingFlags then
        for _, blocked in r.blockingFlags do if blocked then return end end
    end
    r.current = math.min(r.max, r.current + r.regenPerSec * dt)
end
```

One Heartbeat loop ticks every resource on every player:
```lua
RunService.Heartbeat:Connect(function(dt)
    for _, p in Players:GetPlayers() do
        local state = getState(p); if not state then continue end
        Resource.Tick(state.stamina, dt)
        Resource.Tick(state.mana, dt)
        if state.shield then Resource.Tick(state.shield, dt) end
        -- Health regen (special — drives Humanoid):
        local hum = p.Character and p.Character:FindFirstChildOfClass("Humanoid")
        if hum and state.health then
            Resource.Tick(state.health, dt)
            hum.Health = state.health.current
            -- Or: read hum.Health into state.health.current to keep them mirrored
        end
    end
end)
```

For health regen specifically, use the engine's built-in `Humanoid.HealthRegen` if you want the simplest version (regenerates 1% of MaxHealth/sec while alive); for richer rules (no-regen-during-combat), turn the engine's regen off and drive it yourself.

## Default tunings (starting points; tune per game)

| Resource | Max | Regen/sec | Delay (sec) | Notes |
|---|---|---|---|---|
| Health | 100 | 1 (or 5 in casual games) | 5 | Stop regen during combat |
| Stamina | 100 | 25 | 1 | Block while sprinting / dodging |
| Mana | 100 | 5 | 0 (always regen) | Block while channeling |
| Shield (over HP) | 50 | 8 | 4 | Regen kicks back in faster than HP |

For genres where dying is meant to feel costly (RPG hardcore mode, soulslike), regen tunings should make the player commit time to recover. For party/casual games, faster regen keeps the action moving.

## Pool interactions

Real games have rules connecting resources:
- **Sprinting drains stamina, blocks stamina regen.** Stop sprinting → 1s delay → regen resumes.
- **Casting drains mana, blocks mana regen.** Different delay than stamina.
- **Low HP → reduced stamina regen** (you're tired): `regenPerSec * (0.5 + 0.5 * health/maxHealth)`.
- **Shield absorbs damage first** before HP. Shield regenerates faster after no-damage delay.
- **Out of mana? Spell fizzles.** Don't allow an ability to be used at insufficient resource (`combat.md`).

```lua
local function takeDamage(state: CombatState, hum: Humanoid, amount: number)
    if state.shield and state.shield.current > 0 then
        local absorbed = math.min(amount, state.shield.current)
        Resource.Drain(state.shield, absorbed)
        amount -= absorbed
    end
    if amount > 0 then
        Resource.Drain(state.health, amount)
        hum:TakeDamage(amount)
    end
end
```

Centralize via `Combat.ApplyDamage` (see [combat.md](combat.md)) so shield-before-HP is consistent.

## Bar UI primitive

A resource bar is a Frame with a child Frame (the "fill") whose width tracks `current/max`. The basics:

```lua
-- HealthBar, parented under your HUD ScreenGui
local frame = Instance.new("Frame")
frame.Name = "HealthBar"
frame.Size = UDim2.fromOffset(240, 16)
frame.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 4)

local fill = Instance.new("Frame", frame)
fill.Name = "Fill"
fill.Size = UDim2.fromScale(1, 1)               -- starts full
fill.BackgroundColor3 = Color3.fromRGB(80, 220, 80)
Instance.new("UICorner", fill).CornerRadius = UDim.new(0, 4)

local label = Instance.new("TextLabel", frame)
label.BackgroundTransparency = 1
label.Size = UDim2.fromScale(1, 1)
label.TextColor3 = Color3.fromRGB(255, 255, 255)
label.Font = Enum.Font.BuilderSansBold
label.TextSize = 14
label.Text = "100 / 100"
```

To update:
```lua
local function setBar(frame: Frame, current: number, max: number, color: Color3?)
    local fraction = math.clamp(current / max, 0, 1)
    frame.Fill.Size = UDim2.fromScale(fraction, 1)
    if color then frame.Fill.BackgroundColor3 = color end
    frame.TextLabel.Text = string.format("%d / %d", current, max)
end
```

For a smooth fill animation (don't snap on each Heartbeat):
```lua
local TweenService = game:GetService("TweenService")
TweenService:Create(frame.Fill, TweenInfo.new(0.2), {Size = UDim2.fromScale(fraction, 1)}):Play()
```

For 60Hz feel, lerp the fill size in `RenderStepped` toward a target value rather than tweening per change.

## Color by threshold

A health bar that goes green → yellow → red as it depletes is universal:

```lua
local function healthColor(fraction: number): Color3
    if fraction > 0.5 then
        return Color3.fromRGB(80, 220, 80)             -- green
    elseif fraction > 0.25 then
        return Color3.fromRGB(220, 200, 60)            -- yellow
    else
        return Color3.fromRGB(220, 60, 60)             -- red
    end
end
```

For continuous gradient (no banding), interpolate:
```lua
local function gradientColor(fraction: number): Color3
    if fraction > 0.5 then
        return Color3.fromRGB(80, 220, 80):Lerp(Color3.fromRGB(220, 200, 60), (1 - fraction) * 2)
    else
        return Color3.fromRGB(220, 200, 60):Lerp(Color3.fromRGB(220, 60, 60), 1 - fraction * 2)
    end
end
```

## Segmented bars (ammo, charges)

For discrete resources (ammo, ability charges, lives), segments look better than a continuous fill:

```lua
local function buildSegmentedBar(parent: GuiObject, segmentCount: number)
    local list = Instance.new("UIListLayout", parent)
    list.FillDirection = Enum.FillDirection.Horizontal
    list.Padding = UDim.new(0, 2)

    local segments = {}
    for i = 1, segmentCount do
        local seg = Instance.new("Frame", parent)
        seg.Size = UDim2.new(1 / segmentCount, -2, 1, 0)
        seg.BackgroundColor3 = Color3.fromRGB(80, 200, 80)
        table.insert(segments, seg)
    end
    return segments
end

-- Update by toggling segment colors based on count
local function setSegments(segments: {Frame}, filled: number)
    for i, seg in segments do
        seg.BackgroundColor3 = (i <= filled)
            and Color3.fromRGB(80, 200, 80)
            or Color3.fromRGB(60, 60, 60)
    end
end
```

For the gun ammo HUD, segments correspond to bullets in the magazine; reload refills them.

## Damage flash + lag bar

When the player takes damage, a polished UI shows two layers:
- **Front fill** (current health) snaps down immediately
- **"Ghost" fill** behind it fades down slowly so the player sees how much they lost

```lua
local function flashDamage(bar: Frame, oldFraction: number, newFraction: number)
    local ghost = bar:FindFirstChild("Ghost") or Instance.new("Frame", bar)
    ghost.Name = "Ghost"
    ghost.ZIndex = bar.Fill.ZIndex - 1
    ghost.BackgroundColor3 = Color3.fromRGB(220, 60, 60)
    ghost.Size = UDim2.fromScale(oldFraction, 1)
    bar.Fill.Size = UDim2.fromScale(newFraction, 1)

    TweenService:Create(ghost, TweenInfo.new(0.6, Enum.EasingStyle.Quad, Enum.EasingDirection.Out, 0, false, 0.3), {Size = UDim2.fromScale(newFraction, 1)}):Play()
end
```

## Specific resources

### Health (`Humanoid.Health`)
Engine-managed. Read `hum.Health` / `hum.MaxHealth` for bar; modify via `:TakeDamage` / direct assignment. To suppress engine regen, set `hum.HealthDisplayType` and roll your own.

### Stamina / Mana
Per-player state in your CombatState. Drain on action, regen via the standard `Resource.Tick`. UI bars below health.

### Shield / Armor (over HP)
Optional secondary pool. Damage hits Shield first; on depletion, hit HP. Shield regenerates faster but with longer delay. Render as a separate bar above health, or as a layer on top.

### XP
A bar that fills toward "next level." Read from `profile.totalXp` via the level-derivation function in [progression.md](progression.md):
```lua
local level, xpInLevel, xpForNext = levelFromXp(profile.totalXp)
xpBar.Fill.Size = UDim2.fromScale(xpInLevel / xpForNext, 1)
xpBar.Label.Text = string.format("Lv %d  %d / %d", level, xpInLevel, xpForNext)
```
On level-up, animate the bar to 100%, flash, then jump to the next level's empty bar — see "Level-up" in [progression.md](progression.md).

### Ammo
Segmented bar (one segment per bullet) or numeric display. See [guns-fps.md](guns-fps.md) for gun-specific patterns.

### Durability
Per-item attribute (`item.metadata.durability`). Decreases on use; below 0 the item breaks. Show as a small bar in inventory tooltips.

### Hunger / thirst (survival)
Slow drain over time (e.g., -1/min); restore by eating/drinking. Below threshold, periodic damage. Same Resource pattern with very slow regen and depletion.

### Oxygen (underwater / space)
Drains while underwater; restores when surfaced. Below 0, drown damage starts.

## Patterns

### Multi-bar HUD layout
Stack health/shield/stamina/mana vertically in a corner using a `UIListLayout`:
```lua
local container = Instance.new("Frame")
container.Position = UDim2.new(0, 16, 1, -120)
container.Size = UDim2.fromOffset(240, 80)
container.BackgroundTransparency = 1

local list = Instance.new("UIListLayout", container)
list.Padding = UDim.new(0, 4)
list.SortOrder = Enum.SortOrder.LayoutOrder
```

Order matters: most-critical at top (Health → Shield → Stamina → Mana → XP).

### Always-fill-empty (for low max-resource games)
For ability cooldown indicators where "filling" represents readiness:
```lua
local function setCooldown(bar: Frame, remaining: number, total: number)
    local fraction = 1 - (remaining / total)    -- inverse: empty when on cooldown, full when ready
    bar.Fill.Size = UDim2.fromScale(fraction, 1)
end
```

### Critical-state pulse
When a resource is below 25%, gently pulse the bar to draw attention:
```lua
if fraction < 0.25 then
    local pulse = TweenInfo.new(0.5, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true)
    TweenService:Create(bar, pulse, {BackgroundTransparency = 0.4}):Play()
end
```

### Push updates from server, render on client
The server is authoritative for resource values; pushes a snapshot on every change to the owning client:
```lua
-- Server
ResourceUpdateRemote:FireClient(player, {
    health = {cur = state.health.current, max = state.health.max},
    stamina = {cur = state.stamina.current, max = state.stamina.max},
    -- ...
})

-- Client
ResourceUpdateRemote.OnClientEvent:Connect(function(snap)
    setBar(healthBar, snap.health.cur, snap.health.max, healthColor(snap.health.cur / snap.health.max))
    setBar(staminaBar, snap.stamina.cur, snap.stamina.max)
end)
```

For per-frame regen don't push every Heartbeat (network spam) — push on threshold crossings (every 1% change, or when the player takes damage, or every 0.25s).

## Pitfalls

- **Regen running on the client.** Trivial exploit: client claims max stamina forever. Server is authoritative; client predicts.
- **Bar fill exceeding parent bounds.** A `current > max` value renders past the bar. Always clamp in `setBar`.
- **Per-Heartbeat tween creating thousands of Tween objects.** `:Create` allocates. Either lerp manually in RenderStepped or cache one Tween per bar.
- **Bar updates per-frame from server pushes.** Burns network. Push on threshold change, not continuously.
- **Shield bar drawn over health bar but ghost-fade reveals stale state.** Share the ghost-bar between layers carefully or render separate.
- **Color thresholds without a label.** Color-blind players can't tell green from red. Always include the numeric value.
- **Stamina blocks stamina regen but isn't released on flag clear.** Flag-set/unset symmetry — every set of `state.stamina.blockingFlags.sprinting = true` needs a corresponding clear.
- **Health regen during a status effect that should disable it.** Don't forget to add `combat = true` to `state.health.blockingFlags` while in combat (clear after a few seconds of no damage taken).
- **Pool interactions cascading wrongly.** Low-HP-reduces-stamina-regen sounds clever until the player is stuck — can't recover stamina to escape because stamina-regen is gated on health they can't restore. Test cyclic dependencies.
- **Bars render at native resolution → tiny on 4K, huge on phone.** Use Scale-based sizing or `UIScale` keyed off viewport (see [menu-systems.md](menu-systems.md) mobile considerations).
- **Engine `Humanoid.HealthRegen` competing with your custom regen.** Disable the engine's by setting `humanoid.MaxHealth` and overwriting `humanoid.Health` yourself, or by suppressing the regen via custom logic. Don't fight both at once.
- **Resource depletion not blocking action.** "Out of mana" should fail the spell-cast, not silently let it through. Check resource availability *before* applying the cost.
- **Bar going to 0 makes the fill Frame collapse to invisible.** A bar at 0% is a thin line, not nothing — players expect to see "empty." Set `min size 1px` or render an empty-state placeholder.

## See also
- [combat.md](combat.md) — stamina/mana as combat resources; the original location of the regen pattern
- [characters.md](characters.md) — Humanoid.Health, MaxHealth, HealthRegen
- [progression.md](progression.md) — XP bar, level curve
- [status-effects.md](status-effects.md) — effects that modify resource regen rates
- [guns-fps.md](guns-fps.md) — ammo as a segmented bar
- [inventory.md](inventory.md) — item durability as a per-item resource
- [horror-effects.md](horror-effects.md) — sanity meter is a resource bar
- [menu-systems.md](menu-systems.md) — HUD positioning, mobile UI scaling
- [ui-system.md](ui-system.md) — the underlying Frame/UDim2/UIListLayout primitives
- [tweens-animation.md](tweens-animation.md) — fill animations
- [spawn-mechanics.md](spawn-mechanics.md) — resources reset to max at spawn
- [security.md](security.md) — server-authoritative resource state
