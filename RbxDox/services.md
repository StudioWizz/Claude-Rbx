# Services

> Upstream: <https://create.roblox.com/docs/scripting/services>
> Source: `content/en-us/scripting/services.md`

## Purpose
A **service** is a singleton Instance under `game` that exposes engine functionality. You access them via `game:GetService("ClassName")` — never by direct indexing (`game.Players` works for some but is brittle and slow). Caching the reference at the top of every script is the convention.

## The services you'll use constantly

```lua
local Players              = game:GetService("Players")
local ReplicatedStorage    = game:GetService("ReplicatedStorage")
local ServerStorage        = game:GetService("ServerStorage")        -- server only
local ServerScriptService  = game:GetService("ServerScriptService")  -- server only
local RunService           = game:GetService("RunService")
local UserInputService     = game:GetService("UserInputService")     -- client only
local ContextActionService = game:GetService("ContextActionService") -- client only
local TweenService         = game:GetService("TweenService")
local HttpService          = game:GetService("HttpService")
local DataStoreService     = game:GetService("DataStoreService")     -- server only
local MemoryStoreService   = game:GetService("MemoryStoreService")   -- server only
local MessagingService     = game:GetService("MessagingService")     -- server only
local CollectionService    = game:GetService("CollectionService")
local Debris               = game:GetService("Debris")
local SoundService         = game:GetService("SoundService")
local Lighting             = game:GetService("Lighting")
local TextChatService      = game:GetService("TextChatService")
local PathfindingService   = game:GetService("PathfindingService")
local TeleportService      = game:GetService("TeleportService")
local MarketplaceService   = game:GetService("MarketplaceService")
local PhysicsService       = game:GetService("PhysicsService")
local TextService          = game:GetService("TextService")
local Workspace            = game:GetService("Workspace")            -- alias for `workspace`
```

## What each is for (in two lines)

- **Players** — `PlayerAdded`, `PlayerRemoving`, `LocalPlayer` (client), `:GetPlayers()`.
- **ReplicatedStorage / ServerStorage** — containers; see [project-structure.md](project-structure.md).
- **RunService** — heartbeat and render loops, `IsServer/IsClient/IsStudio`. `Heartbeat` (every physics step), `RenderStepped` (client, every frame, before render), `Stepped` (every physics step, before).
- **UserInputService** — keyboard/mouse/touch/gamepad on the client; `InputBegan/Changed/Ended`.
- **ContextActionService** — bind named actions to inputs; preferred over UserInputService for game actions because it's gamepad-aware and reassignable.
- **TweenService** — `:Create(instance, info, goalProps)` for property animations. See [tweens-animation.md](tweens-animation.md).
- **HttpService** — `:GetAsync`, `:PostAsync`, `:JSONEncode`, `:JSONDecode`, `:GenerateGUID`. Server only; must be enabled in game settings.
- **DataStoreService** — persistent per-experience storage. See [datastores.md](datastores.md).
- **MemoryStoreService** — fast cross-server ephemeral storage. See [memorystores.md](memorystores.md).
- **MessagingService** — cross-server pub/sub. See [messaging-http.md](messaging-http.md).
- **CollectionService** — tag Instances and iterate by tag (`:AddTag`, `:GetTagged`, `:GetInstanceAddedSignal`).
- **Debris** — `:AddItem(inst, seconds)` schedules an Instance for destruction.
- **SoundService / Lighting** — global audio mixing and lighting.
- **TextChatService** — modern chat (replaces legacy Chat); use `ChatChannel` and `TextChannel` APIs.
- **PathfindingService** — `:CreatePath(params):ComputeAsync(start, finish)` for NPCs.
- **TeleportService** — move players between places/servers.
- **MarketplaceService** — Robux purchases, gamepasses, devproducts.
- **PhysicsService** — collision groups (`:RegisterCollisionGroup`, `:CollisionGroupSetCollidable`).
- **TextService** — `:FilterStringAsync(text, userId)` for chat/UGC moderation. Required for any user-entered text you display.

## Patterns

### Run on every frame (client)
```lua
RunService.RenderStepped:Connect(function(dt)
    -- update camera, UI, particles
end)
```

### Run on every physics step (server or client)
```lua
RunService.Heartbeat:Connect(function(dt)
    -- update server-authoritative state
end)
```

### Tag-driven setup
```lua
local CollectionService = game:GetService("CollectionService")

local function setupZombie(zombie: Model) ... end

for _, z in CollectionService:GetTagged("Zombie") do setupZombie(z) end
CollectionService:GetInstanceAddedSignal("Zombie"):Connect(setupZombie)
```

## Pitfalls

- **`game.HttpService`** vs **`game:GetService("HttpService")`** — direct indexing fails for services that aren't in the DataModel by default. Always use `GetService`.
- **Calling server-only services from a LocalScript** (DataStore, MemoryStore, Messaging) errors.
- **Filtering text with TextService** is mandatory before displaying any player-entered string. Skipping it can get your experience moderated.

## See also
- [events-signals.md](events-signals.md) — RBXScriptSignal mechanics
- [task-scheduling.md](task-scheduling.md) — `RunService` vs `task.wait`
