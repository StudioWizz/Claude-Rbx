# Scripts & Modules

> Upstream: <https://create.roblox.com/docs/scripting> · <https://create.roblox.com/docs/scripting/module>
> Source: `content/en-us/scripting/module.md`, `content/en-us/scripting/locations.md`

## Purpose
Roblox has three script classes. Choosing the right one is mostly a placement decision, not a code-style decision.

| Class | Where it runs | Where it lives |
|---|---|---|
| `Script` | **Server** (or, since 2022, in any RunContext setting) | `ServerScriptService`, `ServerStorage`, `Workspace` |
| `LocalScript` | **Client** | `StarterPlayerScripts`, `StarterCharacterScripts`, `StarterGui` (under a Player), `StarterPack`, Backpack |
| `ModuleScript` | Wherever it's `require()`d | Anywhere — typically `ReplicatedStorage` (shared) or `ServerStorage` (server-only) |

`Script` instances now also have a `RunContext` property: `Server` (default), `Client`, or `Plugin`. Setting `RunContext = Client` lets a `Script` (not `LocalScript`) run on the client from non-character locations — useful for shared code that needs both contexts.

## ModuleScripts
The reusable unit. A ModuleScript runs **once** per VM the first time it's required; subsequent `require` calls return the cached value.

```lua
-- ReplicatedStorage/Combat/Damage.lua
local Damage = {}

Damage.CRIT_MULTIPLIER = 2

function Damage.compute(base: number, isCrit: boolean): number
    return if isCrit then base * Damage.CRIT_MULTIPLIER else base
end

return Damage
```

```lua
-- Any consumer
local Damage = require(game.ReplicatedStorage.Combat.Damage)
print(Damage.compute(10, true))  -- 20
```

A module **must** end with `return <something>`. Returning a table is the conventional shape; you can return a function or class as well.

### Caching is per-side
Server and client each have their own VM, so the module runs once per side. Don't rely on shared mutable state across the boundary — use Remotes or DataStores for that.

### Class pattern (brief)
```lua
local Player = {}
Player.__index = Player

function Player.new(name: string)
    local self = setmetatable({}, Player)
    self.name = name
    self.hp = 100
    return self
end

function Player:hurt(n: number)
    self.hp -= n
end

return Player
```
Caller: `local p = Player.new("alex"); p:hurt(10)`. For the full treatment — composition over inheritance, `:Destroy()` discipline, forward declarations, when *not* to use OOP — see [oop-patterns.md](oop-patterns.md).

## Plugin scripts
A `Plugin` lives in `~/Documents/Roblox/Plugins` (local) or is published to the Marketplace. It's a Studio-only Script that interacts with the Studio API (selection, undo waypoints, custom widgets). Created via `File → New → Script`, then `Save as Local Plugin`.

## Patterns

### Server entry point
```lua
-- ServerScriptService/Main.server.lua  (Rojo naming convention)
local Players = game:GetService("Players")
local PlayerData = require(game.ServerStorage.PlayerData)

Players.PlayerAdded:Connect(function(player)
    PlayerData.load(player)
end)
```

### Client entry point
```lua
-- StarterPlayerScripts/Bootstrap.client.lua
local UI = require(game.ReplicatedStorage.UI.Main)
UI.mount()
```

## Pitfalls

- **Circular requires.** `A` requires `B`, `B` requires `A` → deadlock at load. Either flatten or inject the dependency at runtime.
- **Mutating module return value.** Whoever requires it first wins; later requirers see the mutation. Treat module exports as read-only after first require, or expose `.new()` factories.
- **`LocalScript` in `Workspace`.** Doesn't run. LocalScripts run only under a Player container or in StarterCharacter.
- **Yielding at the top level of a module.** Every requirer waits. Move yields into functions.
- **Forgetting `return`.** Without it, `require` returns `nil` and you get cryptic indexing errors.

## See also
- [project-structure.md](project-structure.md) — service placement
- [services.md](services.md) — `game:GetService`
- [luau-types.md](luau-types.md) — typed module exports
- [luau-style.md](luau-style.md) — naming, casing, file conventions, Rojo names
- [oop-patterns.md](oop-patterns.md) — class pattern (full treatment)
- [luau-basics.md](luau-basics.md) — language fundamentals the modules are written in
- [community-libraries.md](community-libraries.md) — **Rojo** (on-disk source layout) and **Wally** (package manager)
