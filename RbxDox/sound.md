# Sound & Audio

> Upstream: <https://create.roblox.com/docs/sound> · <https://create.roblox.com/docs/reference/engine/classes/Sound>
> Source: distilled from the Sound, SoundService, and SoundGroup API references

## Purpose
`Sound` is a single audio playback instance. Where you parent a Sound determines its behavior more than any property: under a `BasePart` it's 3D positional; under `SoundService` (or anywhere else) it's 2D and full-volume for everyone; under a player's `PlayerGui` it's that player only. `SoundService` is the global mixer; `SoundGroup` lets you wire multiple Sounds to one volume slider.

## Where parenting matters

| Parent | Behaviour |
|---|---|
| Under a `BasePart` (or `Attachment`) | **3D positional** — distance-attenuated, panned by listener position |
| `SoundService` | **2D global** — every client hears at full volume |
| `Workspace` (loose) | 2D global, same as SoundService |
| Under a `Player.PlayerGui` (or any descendant of a Player) | **2D, only that one player hears it** |
| Under a `Tool` (which is under a Character) | 3D positional from the Tool's position |

This is the single most important rule: **parent under a Part for 3D, parent under SoundService (or PlayerGui for one-player) for 2D**.

## Key API

```lua
-- Properties
sound.SoundId              -- "rbxassetid://1234567890"
sound.Volume               -- 0..10 (1 is unity)
sound.PlaybackSpeed        -- 1 is normal; <1 slower/lower-pitched; >1 faster/higher
sound.Looped               -- bool
sound.PlayOnRemove         -- if true, plays when the Sound is destroyed (one-shot pattern)
sound.TimePosition         -- read/write current playback head
sound.RollOffMode          -- Inverse | Linear | LinearSquare | InverseTapered  (3D attenuation curve)
sound.RollOffMaxDistance   -- studs at which volume reaches near-zero
sound.RollOffMinDistance   -- studs within which volume is full
sound.SoundGroup           -- assign to a SoundGroup for grouped mixing
sound.IsLoaded             -- bool, set after the asset finishes downloading
sound.IsPlaying, IsPaused

-- Methods
sound:Play()
sound:Pause()
sound:Resume()
sound:Stop()

-- Events
sound.Played, Paused, Resumed, Stopped
sound.Ended                -- fires when a non-looping sound finishes
sound.Loaded               -- fires when the asset is ready (use before Play if you need timing precision)
sound.DidLoop              -- each time a looped sound restarts
```

## SoundGroup (mixing)

Group sounds for unified volume control:
```lua
local SoundService = game:GetService("SoundService")

local Master = Instance.new("SoundGroup", SoundService)
Master.Name = "Master"

local Music = Instance.new("SoundGroup", Master); Music.Name = "Music"
local SFX   = Instance.new("SoundGroup", Master); SFX.Name = "SFX"
local UI    = Instance.new("SoundGroup", Master); UI.Name = "UI"

-- Per-sound assignment
buttonClick.SoundGroup = UI
gunshot.SoundGroup = SFX
backgroundTrack.SoundGroup = Music

-- Now setting Music.Volume = 0 mutes all music; Master.Volume scales everything
```

Players' settings menus typically expose Master/Music/SFX sliders that drive these groups.

## Patterns

### One-shot SFX on a UI click
```lua
-- LocalScript
local click = Instance.new("Sound")
click.SoundId = "rbxassetid://9118823106"   -- a click sound
click.Volume = 0.6
click.Parent = SoundService                  -- 2D, this client only when triggered locally

button.Activated:Connect(function() click:Play() end)
```

### 3D ambient loop on an object
```lua
local hum = Instance.new("Sound")
hum.SoundId = "rbxassetid://..."
hum.Looped = true
hum.Volume = 1
hum.RollOffMaxDistance = 40
hum.Parent = generatorPart                   -- 3D, attenuates with distance
hum:Play()
```

### Music with crossfade
```lua
local TweenService = game:GetService("TweenService")
local fade = TweenInfo.new(2)

local function crossfade(from: Sound, to: Sound)
    to.Volume = 0
    to:Play()
    TweenService:Create(from, fade, {Volume = 0}):Play()
    TweenService:Create(to, fade, {Volume = 1}):Play()
    task.delay(2, function() from:Stop() end)
end
```

### Server-triggered sound that all clients hear
```lua
-- Server: place the Sound under a part (3D) or SoundService (2D)
-- Then call :Play() on the server. The IsPlaying property replicates and clients begin playing too.
explosionSound.Parent = explosionPart
explosionSound:Play()
```
You can also use the `PlayOnRemove` trick: parent a Sound configured with `PlayOnRemove = true` under a part, then `Sound:Destroy()` plays it once and cleans up — handy for one-shots without managing the lifecycle.

### Per-player UI sound
```lua
-- Server
local localOnly = Instance.new("Sound")
localOnly.SoundId = "rbxassetid://..."
localOnly.Parent = player.PlayerGui   -- only this player hears it
localOnly:Play()
game:GetService("Debris"):AddItem(localOnly, 5)
```

## SoundService global properties

```lua
SoundService.AmbientReverb       -- Enum.ReverbType — global reverb
SoundService.RespectFilteringEnabled  -- if true, server-side Sound:Play() doesn't replicate to clients (rare; default false)
SoundService.DistanceFactor      -- scales 3D attenuation
SoundService.RolloffScale        -- global rolloff multiplier
SoundService.DopplerScale
```

## Asset upload and SoundId

- Roblox-hosted only — you can't load arbitrary URLs. Upload via Asset Manager or the website.
- After upload, the asset id appears as `rbxassetid://<NUMBER>`.
- All audio must comply with Roblox's audio policy and may be moderated; private uploads are restricted to approved-creators-only by default.
- Cap on simultaneous-playing sounds per client is generous (~hundreds) but each loaded asset uses memory. `Sound:Stop()` doesn't unload; nil-out the SoundId or destroy the Sound to release.

## Pitfalls

- **Parented in the wrong place** is the #1 bug — a 3D sound parented to SoundService plays globally instead of from the world. A 2D click parented to a part only fires for players near it.
- **Server `:Play()` doesn't fire on the server's own ear** because the server has no listener. The clients hear it via property replication.
- **Sound asset not loaded yet** — `:Play()` before `IsLoaded` may delay; if timing matters, wait for `Loaded` first or rely on caching after first play.
- **Volume > 1 distorts.** `Volume = 1` is unity; `2`+ clips. For "louder," use SoundGroup mixing or proximity, not raw Volume.
- **Looped sounds don't fire `Ended`.** They fire `DidLoop` instead.
- **Forgetting to `:Stop()` on cleanup** — a Sound continues playing after its parent is destroyed if you reparent first; always `:Destroy()` the Sound or `:Stop()` then `:Destroy()`.
- **Per-frame `Sound:Play()`** — creates new playback state each call. To restart, prefer `TimePosition = 0; :Play()` or pool sounds.
- **3D rolloff feels wrong** — defaults are conservative. Tune `RollOffMaxDistance` to your scale (Roblox studs are small; for "across a room," 20–40 is realistic).
- **Music interrupted by death/respawn** — if it lives under StarterGui with `ResetOnSpawn = true` (the default for StarterGui descendants), it gets wiped. Parent music under SoundService instead.

## See also
- [services.md](services.md) — SoundService overview
- [tools.md](tools.md) — 3D sounds parented under a Tool's Handle
- [tweens-animation.md](tweens-animation.md) — fade Volume with TweenService
- [vfx.md](vfx.md) — pair sounds with particle effects
- [project-structure.md](project-structure.md) — PlayerGui per-player parenting
