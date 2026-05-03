# Project Structure & DataModel

> Upstream: <https://create.roblox.com/docs/projects/data-model> · <https://create.roblox.com/docs/scripting/locations>
> Source: `content/en-us/projects/data-model.md`, `content/en-us/scripting/locations.md`

## Purpose
Every Roblox experience runs inside the **DataModel** — the tree of services rooted at `game`. Where you put an Instance determines who sees it, when it loads, and what can run from it. Getting placement right is the single most common source of "why doesn't this work?" bugs in Roblox.

## Key services and what goes in them

| Service | Replicates to client? | Use for |
|---|---|---|
| `Workspace` | Yes (3D, physics-simulated) | All physical 3D objects, terrain, the camera |
| `ReplicatedStorage` | Yes (instances visible to both) | Shared ModuleScripts, RemoteEvents/Functions, assets clients need |
| `ReplicatedFirst` | Yes, before everything else | Loading-screen scripts and assets that must run before default UI |
| `ServerStorage` | **No** (server-only) | Server-only assets and ModuleScripts that clients must never see |
| `ServerScriptService` | **No** (server-only) | Server-side `Script` objects |
| `StarterPlayer.StarterPlayerScripts` | Copied into each player on join | Client `LocalScript`s that run regardless of character state |
| `StarterPlayer.StarterCharacterScripts` | Copied into each new Character | Client scripts attached to the character (re-runs on respawn) |
| `StarterGui` | Copied into `Player.PlayerGui` | UI templates (`ScreenGui` instances) |
| `StarterPack` | Copied into `Player.Backpack` | Tools players spawn with |
| `Lighting` | Yes | Time-of-day, atmosphere, post-processing effects |
| `SoundService` | Yes | Global Sound objects, sound groups |
| `Teams` | Yes | Team objects |
| `Chat`, `TextChatService` | Yes | Chat configuration (prefer `TextChatService` — newer) |

## The Player object

When a player joins, the engine creates a `Player` under `Players` with these key children:
- `PlayerGui` — copy of `StarterGui` contents.
- `Backpack` — copy of `StarterPack`; tools here are equippable.
- `PlayerScripts` — copy of `StarterPlayerScripts`.
- `Character` (set asynchronously) — the `Model` in `Workspace` containing the `Humanoid` and body parts.

Wait for `Character` with `player.CharacterAdded:Wait()` or check `player.Character`.

## `WaitForChild` on the client

Replicated descendants may not exist on the first frame on the client even though they exist on the server. Use `:WaitForChild("Name")` instead of direct indexing whenever a `LocalScript` reaches into a replicated container:

```lua
-- BAD on client — Remotes folder may not have replicated yet
local damageRemote = game.ReplicatedStorage.Remotes.Damage   -- nil error on first frame

-- GOOD
local Remotes = game:GetService("ReplicatedStorage"):WaitForChild("Remotes")
local damageRemote = Remotes:WaitForChild("Damage")
```

This is a top-N source of "works in Studio, broken on first join" bugs. If the child genuinely might not arrive (Streaming evicted it, server removed it), pass a timeout: `:WaitForChild("Name", 10)` returns `nil` after the timeout instead of waiting forever.

Server scripts don't need this — the server has the authoritative DataModel and sees everything immediately.

## Patterns

### Place scripts correctly
```lua
-- Server: in ServerScriptService
local Players = game:GetService("Players")
Players.PlayerAdded:Connect(function(player)
    print(player.Name, "joined")
end)
```

```lua
-- Client: in StarterPlayerScripts (LocalScript)
local LocalPlayer = game:GetService("Players").LocalPlayer
LocalPlayer.CharacterAdded:Connect(function(character)
    print("My character spawned")
end)
```

### Share code via ReplicatedStorage
```lua
-- ReplicatedStorage/Shared/Math.lua (ModuleScript)
local Math = {}
function Math.lerp(a, b, t) return a + (b - a) * t end
return Math
```

```lua
-- Any script (server or client)
local Math = require(game:GetService("ReplicatedStorage").Shared.Math)
```

## Pitfalls

- **Putting secrets in `ReplicatedStorage`.** Anything in `ReplicatedStorage` is visible to every client via the explorer — exploit clients can read it. Sensitive assets and server-only modules belong in `ServerStorage`.
- **Server `Script` objects in `Workspace`.** They run, but it's a mess to organize. Keep server scripts in `ServerScriptService`.
- **`LocalScript` in `Workspace`.** Won't run. LocalScripts only execute under a `Player` (PlayerScripts, PlayerGui, Backpack) or in StarterCharacter.
- **Touching `LocalPlayer` on the server.** `Players.LocalPlayer` is `nil` on the server.
- **Forgetting to wait for `Character`.** `player.Character` is `nil` between spawn cycles.

## See also
- [workspace-hierarchy.md](workspace-hierarchy.md) — organising what goes *inside* Workspace (Folder vs Model, parenting rules)
- [client-server-model.md](client-server-model.md) — replication and trust boundaries
- [scripts-and-modules.md](scripts-and-modules.md) — Script vs LocalScript vs ModuleScript
- [services.md](services.md) — `game:GetService` and the most-used services
- [player-lifecycle.md](player-lifecycle.md) — when each container's contents arrive for a player (PlayerAdded → CharacterAdded)
- [tools.md](tools.md), [characters.md](characters.md) — StarterPack/Backpack/Character placement specifics
- [sound.md](sound.md) — PlayerGui parenting for per-player sound
- [teleport.md](teleport.md) — `ReplicatedFirst` for cross-place loading screens
