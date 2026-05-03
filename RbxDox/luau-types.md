# Luau Type Checking

> Upstream: <https://create.roblox.com/docs/luau/type-checking>
> Source: `content/en-us/luau/type-checking.md`, `content/en-us/luau/type-coercion.md`

## Purpose
Luau has an optional, gradual type system. You opt in per-script via a comment on the first line. Strict mode catches whole classes of bugs before runtime — wrong argument types, missing fields, nil dereferences. Use it.

## Modes
First-line comment:
```lua
--!nocheck   -- (default if missing) no analysis at all
--!nonstrict -- analysis runs but only flags clear errors; unknowns are `any`
--!strict    -- analysis runs and unknowns are errors
```
**Recommended:** `--!strict` for ModuleScripts and any non-trivial logic; `--!nonstrict` for quick prototyping scripts.

## Annotations
```lua
--!strict
local x: number = 10
local s: string = "hi"
local maybe: string? = nil          -- optional (string | nil)
local u: number | string = 1        -- union
local tup: {number} = {1, 2, 3}     -- array of number
local map: {[string]: number} = {}  -- map string → number

local function add(a: number, b: number): number
    return a + b
end

local function multi(): (number, string)
    return 1, "ok"
end
```

## Type aliases
```lua
type Vector2D = {x: number, y: number}
type Result<T> = {ok: true, value: T} | {ok: false, err: string}

local function origin(): Vector2D
    return {x = 0, y = 0}
end
```

## Roblox Instance types
The Roblox class names are types. Use them on `WaitForChild`/`FindFirstChild` results.
```lua
local part = workspace:WaitForChild("Lava") :: Part
local hum = character:FindFirstChildOfClass("Humanoid") :: Humanoid?
if hum then hum.Health = 0 end
```
The `::` operator is a **type assertion** — you're telling the checker "trust me, it's this." Use sparingly; prefer narrowing via `if`.

## Narrowing
```lua
local function lengthOf(s: string?): number
    if s == nil then return 0 end
    return #s          -- s is now narrowed to `string`
end

local function describe(v: number | string)
    if type(v) == "number" then
        return v + 1   -- v is `number`
    else
        return #v       -- v is `string`
    end
end
```

## Modules: typed exports
```lua
--!strict
local Math = {}

function Math.lerp(a: number, b: number, t: number): number
    return a + (b - a) * t
end

return Math
```
When required from another script, the returned table carries the inferred function signatures — callers get autocomplete and type errors for free.

## Pitfalls

- **Adding `--!strict` to a legacy script all at once** — you'll drown in errors. Convert one module at a time.
- **Casting away problems with `:: any`.** It silences the checker but doesn't fix the bug. Reach for narrowing first.
- **Forgetting that `WaitForChild` returns `Instance`** in the type system. You must cast or check the class to get specific properties.
- **Cyclic `require`.** Two modules requiring each other will deadlock at load time. Restructure to remove the cycle.
- **Number-vs-int.** Luau's `number` is a double. Use `math.floor`/`//` to coerce when you need integer behaviour.

## See also
- [luau-basics.md](luau-basics.md) — language fundamentals
- [scripts-and-modules.md](scripts-and-modules.md) — module structure
