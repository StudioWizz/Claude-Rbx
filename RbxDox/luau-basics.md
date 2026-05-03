# Luau Basics

> Upstream: <https://create.roblox.com/docs/luau>
> Source: `content/en-us/luau/index.md`, `variables.md`, `tables.md`, `functions.md`, `scope.md`, `operators.md`, `control-structures.md`

## Purpose
Luau is Lua 5.1 with extensions: gradual typing, continue, compound assignment, string interpolation, generalized iteration, and a faster VM. It's the only scripting language Roblox supports. This card covers the syntax most scripts touch every day.

## Variables and types
```lua
local x = 10              -- number
local s = "hello"         -- string
local b = true            -- boolean
local n = nil             -- nil
local t = {1, 2, 3}       -- table (array-like)
local m = {a = 1, b = 2}  -- table (map-like)

local function f() end    -- function value
```
Types: `nil`, `boolean`, `number` (double), `string`, `table`, `function`, `userdata`, `thread`. Roblox types like `Vector3`, `CFrame`, `Color3`, `Instance` are userdata.

`local` is local to the enclosing block. Without `local`, the variable becomes a global — almost always wrong. Roblox sandboxes globals per-script, but you should still avoid them.

## Operators
- Arithmetic: `+ - * / // % ^` (`//` floor div, `^` power)
- Compound: `+= -= *= /= //= %= ^= ..=`
- Comparison: `== ~= < > <= >=`
- Logical: `and`, `or`, `not` (short-circuit; `and`/`or` return values, not booleans)
- String concat: `..`
- Length: `#t` (length of array part of a table or string)

`a or b` returns `a` if truthy, else `b` — the standard "default value" idiom: `local name = config.name or "guest"`.

## Control flow
```lua
if x > 0 then
    print("pos")
elseif x < 0 then
    print("neg")
else
    print("zero")
end

for i = 1, 10 do print(i) end
for i = 10, 1, -1 do print(i) end

for i, v in ipairs(arr) do end       -- ordered, array-style
for k, v in pairs(map) do end        -- all keys, unspecified order
for k, v in t do end                 -- Luau generalized iteration (preferred)

while cond do ... end
repeat ... until cond

for i = 1, 10 do
    if i == 5 then continue end       -- Luau extension
    if i == 8 then break end
end
```

## Functions
```lua
local function add(a, b)
    return a + b
end

-- Multiple returns
local function minmax(t)
    return math.min(table.unpack(t)), math.max(table.unpack(t))
end
local lo, hi = minmax({3, 1, 4, 1, 5})

-- Varargs
local function sum(...)
    local s = 0
    for _, v in {...} do s += v end
    return s
end

-- Methods (colon syntax) — implicit `self`
local Counter = {n = 0}
function Counter:inc() self.n += 1 end
Counter:inc()
```

## Closures and upvalues

A **closure** is a function value that "captures" local variables from the scope it was defined in. The captured variables are called **upvalues** (literally "external local variables") — they're neither global nor local to the closure itself.

```lua
local function makeCounter(start: number)
    local n = start                -- local to makeCounter
    return function()              -- this anonymous function is a closure
        n += 1                     -- n is an UPVALUE here
        return n
    end
end

local next = makeCounter(10)
print(next())   -- 11
print(next())   -- 12
```

Each call to `makeCounter` creates a **separate** `n`. Two counters don't share state:
```lua
local a = makeCounter(0)
local b = makeCounter(100)
print(a(), a(), b())   -- 1, 2, 101
```

Upvalues are how a lot of common Roblox patterns work — debounces, per-player state in handlers, deferred work that "remembers" what it was doing. They're also a source of memory leaks: a closure connected to a long-lived signal keeps every upvalue alive forever.

```lua
RunService.Heartbeat:Connect(function()
    print(part.Position)   -- `part` is an upvalue; this connection holds it alive
end)
-- Even if you Destroy `part` elsewhere, this closure keeps a reference to it.
-- Disconnect the connection (or use a weak reference) to release.
```

See [performance.md](performance.md) for the memory-leak side and [task-scheduling.md](task-scheduling.md) for cleanup patterns.

## Garbage collection and weak tables

Luau is garbage-collected: a value lives as long as something **strongly references** it. When the last strong reference disappears, the GC eventually frees the value.

```lua
local t = {}              -- strong reference to a new table
local also = t            -- another strong reference
t = nil                   -- table still alive — `also` keeps it
also = nil                -- no more strong references — GC will collect
```

The classic Roblox memory leak: a long-lived "cache" table that you keep adding to but never remove from. Even if every other reference to the object goes away, the cache still holds it strongly, so the GC can't collect it.

```lua
local cache = {}
local function getParams(npc)
    if not cache[npc] then
        local p = RaycastParams.new()
        p.FilterDescendantsInstances = {npc.Character}
        cache[npc] = p
    end
    return cache[npc]
end
-- npc:Destroy() elsewhere doesn't remove the cache entry → leak
```

### `__mode` — making references weak

A table can be told to hold its keys, values, or both as **weak references**: still reachable through the table, but invisible to the GC's "is this still in use?" check. When everything else stops pointing at the value, the GC collects it AND removes the entry from the weak table automatically.

You opt in with the `__mode` metamethod:

```lua
local cache = setmetatable({}, {__mode = "k"})   -- weak keys
local cache = setmetatable({}, {__mode = "v"})   -- weak values
local cache = setmetatable({}, {__mode = "kv"})  -- both
```

For the cache above, `__mode = "k"` means "if nothing else holds this `npc` alive, drop the entry."

### **Critical Roblox gotcha: `__mode` does NOT work for Instance keys or values**

> "References to Roblox instances are never weak. Tables that hold such references will never be garbage collected." — Luau docs

Roblox `Instance` userdata is reference-counted by the engine, not the Luau GC. A weak table holding an Instance still holds it strongly from the engine's perspective. So a cache keyed by a Roblox `Instance` (a Part, a Model, a Player) will leak even with `__mode = "k"`.

For Instance-keyed caches you need explicit cleanup:

```lua
-- Option 1: hook the instance's destruction
npc.Destroying:Connect(function() cache[npc] = nil end)

-- Option 2: tie the lifecycle to your own object's :Destroy() and clean the cache there

-- Option 3: don't key by the Instance — key by something the GC owns (a string id, a Lua-side wrapper table)
```

Weak tables work great when the keys/values are pure Luau tables, functions, or threads. They're useful for memoization keyed by Lua objects, observer registries, and similar — but never for "give me the right cleanup for an Instance," where you must use signals like `Destroying`, `AncestryChanged`, or your own teardown.

See [performance.md](performance.md) for the broader memory-leak checklist.

## Constants

Lua and Luau have **no true constants** — every variable is mutable at runtime. The convention is to *name* a variable in `UPPER_SNAKE_CASE` to signal "treat this as read-only":

```lua
local MAX_HEALTH = 100        -- a "constant" by convention; nothing stops you from reassigning
local respawnTimer = 5         -- regular variable
```

Strict-mode type checking won't flag a reassignment of a SCREAMING_SNAKE name. The naming is purely a message to other developers (and future-you). For real immutability of *table contents*, use `table.freeze`:

```lua
local CONFIG = table.freeze({maxPlayers = 10, gameTime = 300})
CONFIG.maxPlayers = 99       -- error: attempt to modify a readonly table
```

`table.freeze` is shallow; nested tables need their own `freeze`. See [luau-style.md](luau-style.md) for the casing convention in full.

## Tables
Arrays use 1-based indexing. The same table can hold both array entries and named keys.

```lua
local t = {}
table.insert(t, "a")        -- append
table.insert(t, 1, "b")     -- insert at index
table.remove(t, 1)          -- remove at index
table.find(t, "a")          -- linear search

table.sort(t, function(a, b) return a < b end)
table.concat({"a","b"}, ",") -- "a,b"

local copy = table.clone(t) -- shallow copy
table.freeze(copy)          -- immutable
```

For deep work, `table.clone` is shallow; either write a recursive deep-copy or restructure to avoid needing one.

## Error handling

Luau has two error mechanisms: **raising** an error (`error()`, `assert()`) and **catching** one (`pcall`, `xpcall`). The convention in Roblox code is to use exceptions sparingly — for "this should never happen" programmer mistakes — and to use **`success, value` returns** for "this might fail at runtime."

### Raising
```lua
error("bad input")                    -- raises immediately, propagates up
error("bad input", 2)                 -- 2 = blame the caller, not this line
assert(typeof(x) == "number", "x must be a number")  -- error if condition false
```
`assert(cond, msg)` is the idiomatic way to validate function arguments — fails fast with a clear message.

### Catching
```lua
local ok, result = pcall(function()
    return riskyThing()
end)
if not ok then
    warn("riskyThing failed:", result)
    return
end
-- use `result`
```

`xpcall(fn, handler, ...)` lets you supply an error handler that runs with the stack still live (useful for capturing `debug.traceback()`).

### Convention: return `success, value` instead of throwing

For functions that fail in expected ways (network down, item not found, validation failed), prefer the multi-return shape over `error`:

```lua
-- Function design
local function loadProfile(userId: number): (boolean, any)
    local ok, data = pcall(function() return store:GetAsync("U_" .. userId) end)
    if not ok then return false, data end          -- data is the error message
    return true, data or defaultProfile()
end

-- Caller
local ok, profile = loadProfile(player.UserId)
if not ok then
    warn("could not load profile:", profile)
    return
end
-- profile is the value
```

This matches the shape of `pcall` itself, the shape Roblox engine APIs use for many operations, and removes the need for the caller to wrap your function in another `pcall`.

### When to use which

| Situation | Use |
|---|---|
| Programmer mistake (bad arg type, invariant violated) | `assert` / `error` — fail fast and loud |
| Expected runtime failure (network, missing data, race) | Return `false, errMsg` |
| Wrapping a Roblox `*Async` call | `pcall` + retry-with-backoff (see [datastores.md](datastores.md)) |
| Composing many async calls with cleanup | Promise library — see [community-libraries.md](community-libraries.md) |

### Async functions and yielding errors

Roblox `*Async` calls (DataStore, HttpService, MarketplaceService, PathfindingService) yield AND can raise. Always `pcall` them:

```lua
local ok, data = pcall(function() return store:GetAsync(key) end)
```

A bare `store:GetAsync(key)` will crash the calling thread on any network blip. See [task-scheduling.md](task-scheduling.md) for what "crash the thread" means in Roblox and how it doesn't take down the whole script.

## Strings
```lua
local s = "hello"
#s                          -- length
s:upper(); s:lower()
s:sub(1, 3)                 -- "hel"
s:find("ll")                -- 3, 4
s:gsub("l", "L")            -- "heLLo", 2
string.format("x=%d", 42)
`x = {x}, name = {s}`       -- string interpolation (Luau)
```

## Pitfalls

- **Forgetting `local`.** Creates an implicit global. In strict mode the type checker flags this.
- **Using `wait()` / `spawn()` / `delay()`.** These are deprecated. Use `task.wait`, `task.spawn`, `task.defer`, `task.delay` — see [task-scheduling.md](task-scheduling.md).
- **`pairs` vs `ipairs` vs generalized.** `ipairs` stops at the first `nil`. `pairs` iterates all keys but in unspecified order. Luau's `for k, v in t do` does the right thing for arrays and maps and is the modern default.
- **Truthiness.** Only `false` and `nil` are falsy. `0` and `""` are truthy.
- **`#t` with holes.** If your "array" has nil gaps, `#t` is undefined. Keep arrays dense or use a separate `count` field.

## See also
- [luau-style.md](luau-style.md) — naming, casing, file conventions, idioms
- [luau-types.md](luau-types.md) — gradual type system
- [task-scheduling.md](task-scheduling.md) — modern async primitives
- [scripts-and-modules.md](scripts-and-modules.md) — where Luau code lives
