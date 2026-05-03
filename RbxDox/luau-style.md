# Luau Style & Naming

> Upstream: <https://roblox.github.io/lua-style-guide/> (Roblox's official Luau style guide)
> Source: distilled from the Roblox Luau Style Guide and conventions used throughout the official creator-docs examples

## Purpose
Roblox/Luau has well-established naming and style conventions. Following them makes your code blend in with every Roblox sample, plugin, and open-source library — and the Luau type checker, the Script Editor's autocomplete, and Rojo all have implicit assumptions baked around them.

## Casing rules at a glance

| Thing | Convention | Example |
|---|---|---|
| Roblox API (classes, properties, methods, events, Enums) | **PascalCase** | `BasePart.CFrame`, `Humanoid:TakeDamage()`, `Enum.Material.Plastic` |
| Services | **PascalCase** (their class name) | `Players`, `ReplicatedStorage`, `RunService` |
| Instance names in the DataModel | **PascalCase** | `Workspace.Baseplate`, `ReplicatedStorage.Remotes.Damage` |
| Public module fields (functions, constants on the returned table) | **PascalCase** | `Math.Lerp`, `Combat.MAX_RANGE` |
| Private module fields (helpers not on the returned table) | **camelCase** with optional `_` prefix | `local function _validate(...)` |
| Local variables | **camelCase** | `local playerCount = 0` |
| Function parameters | **camelCase** | `function damage(player, amount)` |
| Module-level constants (Lua has no true constants — naming is a convention; pair with `table.freeze` for table immutability) | **UPPER_SNAKE_CASE** | `local MAX_HEALTH = 100` |
| Type aliases | **PascalCase** | `type PlayerData = {coins: number}` |
| File / ModuleScript names | **PascalCase**, matches the returned value | `PlayerData.lua` returns `PlayerData` table |

The single most-violated rule: **Instance names in the DataModel are PascalCase**. `workspace.MyTower` not `workspace.myTower`. The Roblox engine itself uses PascalCase for everything it ships, so anything camelCase in the tree stands out as user code and breaks the visual consistency of the Explorer.

### Acronyms in names
Treat multi-letter acronyms as **one word**. Capitalize only the first letter:
```lua
-- Good
local jsonValue = ...
function MakeHttpCall() end
local idToken = ...

-- Bad
local JSONValue = ...
function MakeHTTPCall() end
local IDToken = ...
```
Exception: when the acronym is a *set of distinct things*, capitalize all of it: `anRGBValue`, `getXYZ`. (Rare — when in doubt, treat as one word.)

## Naming patterns for clarity

### Booleans — start with `is`, `has`, `can`, `should`, `was`
```lua
local isReady = true
local hasFinished = false
local canAttack = checkRange(target)
local shouldRespawn = lives > 0
```
Avoid bare adjectives (`local ready = true`) — at the call site you can't tell if it's a flag or a value.

### Predicate functions — same prefixes
```lua
local function isAlive(character: Model): boolean
    local hum = character:FindFirstChildOfClass("Humanoid")
    return hum ~= nil and hum.Health > 0
end
```

### Event handlers — `on` + event name
```lua
local function onPlayerAdded(player: Player) ... end
Players.PlayerAdded:Connect(onPlayerAdded)
```

### Verb-first methods
```lua
function Inventory:AddItem(item) end       -- not :ItemAdd, not :Item
function Inventory:RemoveItem(item) end
function Inventory:HasItem(item) end       -- predicate exception
```

### Counts and indices
- `count` for totals: `playerCount`
- `index` (or `i` in tight loops) for positions
- Loop variable `_` when unused: `for _, player in Players:GetPlayers() do ... end`

### Yielding functions — `Async` suffix
The Roblox engine itself follows a strong convention: **functions that yield end with `Async`** (`HttpService:GetAsync`, `DataStore:UpdateAsync`, `TextService:FilterStringAsync`, `PathfindingService:ComputeAsync`). Adopt this for your own code:

```lua
function PlayerData.LoadAsync(player: Player): PlayerData    -- yields (DataStore call)
function PlayerData.GetCached(userId: number): PlayerData?    -- doesn't yield
```

Callers can then tell at the call site whether they need to be inside a coroutine, wrap in `task.spawn`, or guard against re-entrancy. Save `pcall` discipline for `Async` functions, since those are the ones that can fail on network errors.

### Explicit nil checks
Luau treats `false` and `nil` as the only falsy values, but plain `if x then` doesn't distinguish "x is false" from "x is nil" from "x doesn't exist." For optionals, prefer **explicit** comparison:

```lua
if humanoid then ... end           -- ambiguous: false? nil? present?
if humanoid ~= nil then ... end    -- clearly: "is it present?"

if isReady == true then ... end    -- clearly: "is it specifically true?"
```

Style preference, not a hard rule. Stay consistent within a file. The exception is `if-expression` (`local x = if cond then a else b`), where the truthy form is fine.

## File / script naming

- **ModuleScript** name = name of the value it returns. `PlayerData.lua` returns `local PlayerData = {} ... return PlayerData`. The `require` site reads `local PlayerData = require(...)` — no rename, no confusion.
- **Script** (server) and **LocalScript** (client) names describe what they *do*: `Bootstrap`, `RoundManager`, `CameraController`. They don't return values.
- **Rojo file naming convention** (if syncing from disk):
  - `Foo.lua` → ModuleScript
  - `Foo.server.lua` → Script (server)
  - `Foo.client.lua` → LocalScript (client)
  - `init.lua` / `init.server.lua` / `init.client.lua` → the script *becomes* its parent folder; useful for grouping a module + its sub-modules.

## Module structure idioms

```lua
--!strict
-- 1. Services first, cached at the top
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- 2. Module imports next
local Damage = require(ReplicatedStorage.Combat.Damage)

-- 3. Constants
local MAX_HEALTH = 100
local RESPAWN_DELAY = 5

-- 4. Type aliases
type State = "Idle" | "Combat" | "Dead"

-- 5. Module table
local Combat = {}

-- 6. Private helpers (lowercase, optionally _-prefixed)
local function _validateTarget(target: Instance): boolean
    return target:IsA("Model") and target:FindFirstChildOfClass("Humanoid") ~= nil
end

-- 7. Public API on the module table
function Combat.Attack(attacker: Player, target: Model)
    if not _validateTarget(target) then return end
    Damage.Apply(target, 25)
end

return Combat
```

This top-to-bottom order (services → requires → constants → types → module → private → public → return) is what every well-maintained Roblox open-source project follows. Readers know where to look.

## Idioms and small rules

- **`local function name()` over `name = function()`** — same effect, but `local function` allows recursion (`local function f() f() end` works; the assignment form doesn't see itself).
- **Early-return for guards.** No deep `if`-pyramids:
  ```lua
  -- BAD
  if player then
      if player.Character then
          if player.Character:FindFirstChild("Humanoid") then
              -- do thing
          end
      end
  end
  -- GOOD
  if not player then return end
  local char = player.Character
  if not char then return end
  local hum = char:FindFirstChild("Humanoid")
  if not hum then return end
  -- do thing
  ```
- **Always `local`.** A bare `x = 5` makes `x` global — almost never what you want. Strict mode flags this.
- **`task.wait` not `wait`**, `task.spawn` not `spawn`, `task.defer` for end-of-step. See [task-scheduling.md](task-scheduling.md).
- **`if-expression` for ternary (Luau extension)**, NOT the old `and-or` trick:
  ```lua
  -- GOOD
  local sign = if x >= 0 then 1 else -1
  -- BAD: silently wrong if the "then" value is itself falsy
  local sign = x >= 0 and 1 or -1
  ```
- **String interpolation (Luau extension)** with backticks:
  ```lua
  print(`Player {player.Name} has {coins} coins`)
  ```
- **Compound assignment:** `coins += 10` (also `-=`, `*=`, `/=`, `..=`).
- **Generalised iteration:** `for k, v in t do` works for both arrays and maps. Prefer over `pairs` / `ipairs`.
- **Continue:** `if skip then continue end` — Luau extension, no need for nested `if not skip then ... end`.
- **`do` blocks for scope.** Limit a helper variable's lifetime to a closure:
  ```lua
  local nextId
  do
      local last = 0
      nextId = function() last += 1; return last end
  end
  ```

## Formatting

- **Indent with tabs**, displayed as 4 columns. (Roblox's official guide is explicit about tabs over spaces.)
- **Line length** target ~100 columns; comments wrap to 80.
- **No semicolons.** Lua doesn't need them and the convention is to omit. One statement per line.
- **No trailing whitespace.** End every file with a single newline.
- **Double quotes for strings**, unless the string contains double quotes and you'd rather not escape:
  ```lua
  print("hello")
  print('she said "hi"')   -- single quotes OK to avoid escaping
  ```
- **Always parens when calling a function.** Lua's no-paren sugar (`f"x"`, `f{a=1}`) is allowed by the language but disallowed by the style:
  ```lua
  -- GOOD
  print("hello")
  build({size = 4})
  -- BAD
  print "hello"
  build{size = 4}
  ```
- **Trailing comma in multi-line tables**:
  ```lua
  local config = {
      maxHealth = 100,
      respawnTime = 5,    -- trailing comma — makes diffs cleaner
  }
  ```
- **Block-opening syntax goes inline**, not on its own line:
  ```lua
  -- GOOD
  local t = {
      a = 1,
  }
  if cond then
      doThing()
  end
  -- BAD
  local t =
  {
      a = 1,
  }
  ```
- **Multi-line conditions: `if` and `then` on their own lines**, operators at line start:
  ```lua
  if
      someReallyLongCondition
      and someOtherCondition
      and yetAnother
  then
      doSomething()
  end
  ```
- **No parentheses around `if` / `while` / `repeat` conditions.** They're not required and look out of place: `if cond then`, not `if (cond) then`.
- **Function arguments**: keep small (1–2 preferred). If a call needs more, pass a single options table.
- **Sort `require` calls alphabetically** within their group (services, then external packages, then same-project modules).
- **One blank line** to group related code; never start a block with a blank line.

## Comments

- Default to **none**. Self-documenting names beat comments.
- When you do comment, explain **why**, not what. `-- token must be base64 (Discord webhook quirk)` is useful; `-- increment counter` is noise.
- One short line. Multi-paragraph block comments don't pull their weight in scripts.
- For module-level documentation of a public API, put a one-line comment above the function — not a Doxygen-style block.

## What NOT to do

- **Hungarian notation:** `tblPlayers`, `strName`, `iCount`. Type checker handles this.
- **Single-letter names beyond loop indices.** `i`, `j`, `k` for loops are fine; `local p = ...` for a player is not.
- **Mixing casing within one file.** Pick a side per identifier kind and stay consistent.
- **`MyClass_DoThing()` snake-mixed-with-Pascal.** Stick to the conventions.
- **Names that lie.** `processData` that also writes a DataStore should be `processAndSaveData` or split into two functions.
- **Globals for "convenience."** Use module returns and `require`.
- **Re-using variable names by shadowing**:
  ```lua
  for _, player in Players:GetPlayers() do
      local player = player.Character    -- DON'T — shadowing the loop var with a different concept
  end
  ```

## Common Roblox-specific naming examples

```lua
-- Caching services
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")

-- Local references (camelCase)
local localPlayer = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- Constants (UPPER_SNAKE)
local MAX_HEALTH = 100
local SPAWN_POSITION = Vector3.new(0, 50, 0)

-- Module table (PascalCase, matches filename)
local PlayerData = {}

-- Public API (PascalCase methods on the module)
function PlayerData.Load(player: Player) end
function PlayerData.Save(player: Player) end

-- Predicate (camelCase, prefixed)
local function isAdmin(player: Player): boolean
    return ADMIN_IDS[player.UserId] == true
end

-- Handler (camelCase, on-prefixed)
local function onPlayerAdded(player: Player) end

-- Instance names in the tree (PascalCase)
ReplicatedStorage.Remotes.RequestPurchase
ServerStorage.Modules.PlayerData
Workspace.Map.Spawns.RedTeam
```

## See also
- [luau-basics.md](luau-basics.md) — language syntax
- [luau-types.md](luau-types.md) — type annotations and `--!strict`
- [scripts-and-modules.md](scripts-and-modules.md) — module structure and Rojo file naming
