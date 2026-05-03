# Horror Effects

> Source: distilled from horror / survival-horror Roblox patterns

## Purpose
Horror games rely on **mood manipulation** — sanity meters, screen-effect distortions on monster proximity, jumpscares, flashlights with limited battery, ambient dread audio. The visuals are mostly compositions of `lighting.md` post-effects, `sound.md` 3D positional audio, `vfx.md` particles, and `tweens-animation.md` for the timing — but the *patterns* (sanity drain, jumpscare framework, flashlight tool) are genre-specific enough to warrant their own card. Niche compared to the other gameplay cards, but if you're building horror, this is the assembly.

## Sanity meter

A per-player value that drains in scary situations and restores in safe ones. Drives screen effects, audio, gameplay penalties.

```lua
-- Profile field
profile.sanity = 100         -- 0..100

local Sanity = {}

function Sanity.Drain(player: Player, amount: number)
    local profile = PlayerData.Get(player); if not profile then return end
    profile.sanity = math.max(0, profile.sanity - amount)
    Sanity.PushUpdate(player, profile.sanity)
end

function Sanity.Restore(player: Player, amount: number)
    local profile = PlayerData.Get(player); if not profile then return end
    profile.sanity = math.min(100, profile.sanity + amount)
    Sanity.PushUpdate(player, profile.sanity)
end

function Sanity.PushUpdate(player: Player, value: number)
    SanityRemote:FireClient(player, value)
end

return Sanity
```

### Drain triggers
- **Monster line-of-sight**: per-frame raycast from monster to player; if visible, drain at a rate. See [raycasting.md](raycasting.md).
- **In darkness without flashlight**: check Lighting / nearby light sources.
- **Witnessing events**: corpse, monster kill, environmental scare.

```lua
-- Per-Heartbeat (server, throttled)
RunService.Heartbeat:Connect(function(dt)
    for _, p in Players:GetPlayers() do
        if isMonsterVisibleTo(p) then Sanity.Drain(p, 5 * dt) end
        if isInDarkness(p) then Sanity.Drain(p, 2 * dt) end
    end
end)
```

### Restore triggers
- **Safe rooms** (tagged regions). Detect via overlap query.
- **Sanctity items** (light a candle, reach a checkpoint).
- **Time** (slow passive recovery in fully-lit areas).

### Sanity-driven effects
The client maps `sanity` to severity of effects:

```lua
-- LocalScript
local cc = Lighting:FindFirstChildOfClass("ColorCorrectionEffect")
local blur = Lighting:FindFirstChildOfClass("BlurEffect")
local heartbeatSound = SoundService.Heartbeat

local function applySanityEffects(value: number)
    local severity = (100 - value) / 100        -- 0 (sane) → 1 (insane)
    cc.Saturation = -severity * 0.7
    cc.Contrast = severity * 0.3
    blur.Size = severity * 8

    heartbeatSound.Volume = severity * 0.6
    heartbeatSound.PlaybackSpeed = 1 + severity * 0.5

    if value < 20 then
        triggerHallucinationFx()        -- random fake monster shadows, whispers, etc.
    end
end

SanityRemote.OnClientEvent:Connect(applySanityEffects)
```

See [lighting.md](lighting.md) for the post-effect surface.

## Monster proximity distortion

Independent of sanity, distort the screen when a monster is close. The effect should escalate with proximity.

```lua
-- LocalScript
local function updateProximityFx()
    local nearestDist = math.huge
    for _, monster in CollectionService:GetTagged("Monster") do
        local hrp = monster:FindFirstChild("HumanoidRootPart")
        if hrp then
            local d = (hrp.Position - LocalPlayer.Character.HumanoidRootPart.Position).Magnitude
            nearestDist = math.min(nearestDist, d)
        end
    end

    local intensity = math.clamp(1 - nearestDist / 50, 0, 1)
    chromaticAberrationCC.TintColor = Color3.fromRGB(255 - intensity * 100, 255 - intensity * 200, 255)
    cameraShakeAmount = intensity * 0.3
end

RunService.RenderStepped:Connect(updateProximityFx)
```

For camera shake, jitter `Camera.CFrame` per frame by `cameraShakeAmount * Vector3.new(noise, noise, noise)`.

## Jumpscare framework

A jumpscare is a coordinated burst: visual flash + loud sound + camera punch + monster flash on screen.

```lua
local function jumpscare(player: Player, monsterId: string)
    JumpscareRemote:FireClient(player, {monsterId = monsterId})
end

-- Client
JumpscareRemote.OnClientEvent:Connect(function(data)
    -- 1. Loud stinger sound
    local stinger = Instance.new("Sound")
    stinger.SoundId = JUMPSCARE_SOUND_BY_MONSTER[data.monsterId]
    stinger.Volume = 4
    stinger.Parent = SoundService
    stinger:Play()
    Debris:AddItem(stinger, 2)

    -- 2. Full-screen monster image flash
    local screen = LocalPlayer.PlayerGui:WaitForChild("JumpscareGui")
    local image = screen.Image
    image.Image = JUMPSCARE_IMAGE_BY_MONSTER[data.monsterId]
    image.ImageTransparency = 0
    image.Visible = true
    TweenService:Create(image, TweenInfo.new(1.5, Enum.EasingStyle.Quad), {ImageTransparency = 1}):Play()
    task.delay(1.5, function() image.Visible = false end)

    -- 3. Camera shake + brief blur
    triggerCameraShake(0.5, 0.5)
    local blur = Lighting:FindFirstChildOfClass("BlurEffect")
    blur.Size = 16
    TweenService:Create(blur, TweenInfo.new(1), {Size = 0}):Play()

    -- 4. Sanity drain
    -- (server-side, fires in parallel)
end)
```

Server fires the jumpscare when the monster's logic detects "got close enough to player without being seen first." Use sparingly — overuse desensitizes.

## Flashlight tool

The classic horror tool: limited cone of vision, drains battery, can be toggled.

```lua
-- LocalScript inside the Flashlight Tool
local tool = script.Parent
local light: SpotLight = tool.Handle.Light    -- pre-authored SpotLight
local battery = 100      -- seconds of usage remaining
local active = false

local function setActive(state: boolean)
    if state == active then return end
    active = state
    light.Enabled = active and battery > 0
    FlashlightRemote:FireServer(state)        -- mirror to server (other clients see it lit)
end

tool.Activated:Connect(function() setActive(not active) end)

RunService.Heartbeat:Connect(function(dt)
    if active and battery > 0 then
        battery = math.max(0, battery - dt)
        if battery == 0 then setActive(false) end
    end
end)

-- Battery pickup increases battery
PickupBatteryRemote.OnClientEvent:Connect(function(amount)
    battery = math.min(100, battery + amount)
end)
```

For server-side validation (anti-cheat infinite battery), the server tracks battery too and pushes it to the client. Client UI displays a battery bar.

For **directional darkness** — only the cone of the SpotLight is bright, everything else is dark — turn ambient and outdoor ambient way down in `Lighting`:
```lua
Lighting.Ambient = Color3.new(0.05, 0.05, 0.05)
Lighting.OutdoorAmbient = Color3.new(0.05, 0.05, 0.05)
Lighting.Brightness = 0      -- no global sun
```

## Ambient dread audio

A constant low-volume drone that ramps with tension:
```lua
local drone = Instance.new("Sound")
drone.SoundId = "rbxassetid://..."
drone.Looped = true
drone.Volume = 0.1
drone.Parent = SoundService
drone:Play()

-- Ramp volume with proximity / sanity
RunService.Heartbeat:Connect(function()
    local tensionLevel = computeTension()      -- 0..1
    drone.Volume = 0.1 + tensionLevel * 0.5
    drone.PlaybackSpeed = 1 + tensionLevel * 0.3   -- slight pitch-up for unease
end)
```

Random sound stings (sudden whisper, footsteps, distant scream) every 30–90s reinforce dread without scripting individual events. See [sound.md](sound.md).

## Hide-and-seek mechanic

Player presses a key to "hide" inside a closet / under bed:
```lua
HideRemote.OnServerEvent:Connect(function(player, hidingSpot)
    if not isValidHidingSpot(hidingSpot, player) then return end
    -- Anchor player inside the spot
    local hrp = player.Character.HumanoidRootPart
    hrp.CFrame = hidingSpot.CFrame
    hrp.Anchored = true
    -- Mark player as hidden — monsters' AI checks this and skips them
    player:SetAttribute("Hidden", true)
end)
```

Monster AI checks player's `Hidden` attribute; if true, ignores the player as a target.

## Pitfalls

- **Sanity drain too aggressive.** Players hit 0 in 30 seconds and the effects become normalized. Tune to several minutes of cumulative exposure.
- **Jumpscare overuse.** First jumpscare is scary; tenth is annoying. Cap to 1–2 per session, or escalate sparingly.
- **Loud stinger sound at Volume = 10.** Real-world hearing damage. Cap at Volume = 4 or so even for jumpscares.
- **Screen effects applied server-side.** Lighting changes server-side affect every player. For per-player effects, parent the post-effect under the player's `PlayerGui` or only set them client-side.
- **Flashlight battery purely client-side.** Trivial to mod for infinite. Track server-side too; client predicts.
- **Camera shake on `Heartbeat` rather than `RenderStepped`.** Jitter looks laggy. Use RenderStepped for any visual.
- **Hallucination FX revealing real monsters.** If the same model spawns for hallucination as for real, players learn to ignore both. Use distinct visual/audio cues for the fake ones.
- **Ambient drone playing through main menu.** Disable horror audio in lobby/safe areas; only in horror zones.
- **Server doesn't know about jumpscare events.** Then server-side "drain sanity on jumpscare" doesn't fire. Server fires the jumpscare; client renders it.
- **Hide attribute not cleared on unhide.** Player stays "hidden" forever; monsters never target them. Clear on move/unhide event.
- **Monster proximity distortion drawing every frame across hundreds of monsters.** Cap to nearest-3 or use a tagged-list cache.

## See also
- [lighting.md](lighting.md) — ColorCorrection, Blur, Atmosphere for mood
- [sound.md](sound.md) — 3D positional ambient and stings
- [vfx.md](vfx.md) — particle effects (mist, blood)
- [npcs.md](npcs.md) — monster NPC composition; AI that respects `Hidden` attribute
- [tools.md](tools.md) — flashlight as a Tool
- [tweens-animation.md](tweens-animation.md) — timing the jumpscare flash
- [raycasting.md](raycasting.md) — line-of-sight check for sanity drain
- [tags.md](tags.md) — tag monsters, safe rooms, hiding spots
- [datastores.md](datastores.md) — persist sanity if it survives sessions (most horror games reset)
