# OOP & Class Patterns

> Upstream: <https://create.roblox.com/docs/luau/metatables> · <https://roblox.github.io/lua-style-guide/#oop>
> Source: distilled from the Luau metatables docs and idiomatic class patterns across the Roblox open-source ecosystem (Knit, Roact, ProfileService, etc.)

## Purpose
Luau has no `class` keyword. Object-oriented code is built on top of tables and **metatables** — specifically the `__index` metamethod. The convention is well-established and consistent across the Roblox ecosystem; learning the one canonical class pattern unlocks reading and writing 90% of Roblox community libraries.

## The canonical class pattern

```lua
--!strict
local Counter = {}
Counter.__index = Counter

export type Counter = typeof(setmetatable({} :: {
    n: number,
    step: number,
}, Counter))

function Counter.new(start: number?, step: number?): Counter
    local self = setmetatable({}, Counter)
    self.n = start or 0
    self.step = step or 1
    return self
end

function Counter:Increment(): number
    self.n += self.step
    return self.n
end

function Counter:Reset()
    self.n = 0
end

function Counter:Destroy()
    -- disconnect anything, release references
end

return Counter
```

Caller:
```lua
local Counter = require(game.ReplicatedStorage.Counter)
local c = Counter.new(10, 5)
print(c:Increment())   -- 15
print(c:Increment())   -- 20
```

### Why this works
- The module returns a **plain table** with the class's methods on it.
- `Counter.__index = Counter` makes the class table double as the metatable's lookup target. When you do `c:Increment()`, Lua finds `Increment` not on `c` itself but on `c`'s metatable's `__index` (= the `Counter` table).
- `Counter.new` is a **factory** (dot-syntax, not colon). It builds an empty table, attaches the metatable, fills in instance fields, returns it.
- Methods use **colon syntax** (`function Counter:Method()`) — that's just sugar for `function Counter.Method(self, ...)`.

### `:Destroy()` convention

Always provide a `:Destroy()` (or `:Disconnect()`) method on classes that hold connections, Instances, or threads. Roblox developers expect to be able to call it:
```lua
c:Destroy()
c = nil
```

Inside `Destroy`, disconnect every signal, `:Destroy()` any owned Instances, cancel any spawned threads. Without this, your class leaks. See [luau-basics.md → Garbage collection and weak tables](luau-basics.md).

## Composition over inheritance

The Roblox idiom is to **avoid class inheritance** and instead build complex objects by **composing simpler ones as fields** ("components").

```lua
-- Instead of: Vehicle ← Car ← SportsCar (inheritance chain)
-- Do this:

local function newSportsCar(model: Model): SportsCar
    return {
        body = Body.new(model.Chassis),         -- component
        engine = Engine.new(model.Engine),       -- component
        wheels = Wheels.new(model.Wheels),       -- component
        boost = TurboBoost.new(),                -- component
    }
end

function SportsCar:Accelerate(dt: number)
    self.engine:ApplyThrottle(1)
    self.boost:Tick(dt)
    self.wheels:Spin(self.engine.rpm)
end
```

**Why composition wins in Roblox:**
- Instances themselves are already a parent/child tree (Models contain Parts contain Attachments). Adding a class hierarchy on top duplicates that structure.
- Roblox systems (Tools, Humanoids, Animators) are designed to be *attached* to characters, not subclassed.
- Inheritance chains in Luau are fragile — `__index` chains get long, debugging is harder, and the type checker handles them poorly.
- Components can be added/removed at runtime; subclasses can't.

### When inheritance IS okay

For 1 level of "specialisation" of a small base class — e.g., a `BaseUI` with `Button` and `Toggle` subclasses — inheritance is fine. Beyond one level, refactor to composition.

```lua
local Toggle = setmetatable({}, {__index = BaseUI})  -- inherit from BaseUI
Toggle.__index = Toggle
```

The `setmetatable({}, {__index = BaseUI})` chain makes `Toggle` look up missing keys on `BaseUI`. Each instance still has `Toggle` as its direct metatable.

## Single class per module

One ModuleScript = one class. The file is named after the class (`Counter.lua` returns `Counter`). This:
- Matches the `require(...)` site naming exactly.
- Keeps each file small and easy to test.
- Lets Studio's "go to definition" jump straight to the class.

If you find yourself defining two classes in one module, split the file.

## Forward declarations

Lua resolves names at function-call time, not at definition time, but **only for things in scope**. Two functions in the same file that need to call each other need a forward declaration:

```lua
-- WRONG — `bar` doesn't exist yet when `foo` is defined
local function foo()
    bar()
end

local function bar()
    foo()
end

-- RIGHT — declare `bar` first, assign later
local bar    -- forward declaration

local function foo()
    bar()    -- works: `bar` is in scope, currently nil, will be set before foo is called
end

bar = function()
    foo()
end
```

Same trick for module-table methods that internally reference each other before all are defined.

## Member naming on classes

- **Public members and methods**: PascalCase (`self.Health`, `:TakeDamage()`).
- **Private members and methods**: prefix with `_` (`self._connections`, `:_internalUpdate()`). The `_` is a hint to other developers; Luau doesn't enforce it.

```lua
function Vehicle.new()
    local self = setmetatable({}, Vehicle)
    self.Speed = 0                       -- public
    self._connections = {}               -- private
    return self
end

function Vehicle:_addConnection(conn)    -- private
    table.insert(self._connections, conn)
end

function Vehicle:Destroy()               -- public
    for _, c in self._connections do c:Disconnect() end
    table.clear(self._connections)
end
```

See [luau-style.md](luau-style.md) for the broader naming conventions.

## Other useful metamethods

`__index` is the workhorse, but a couple of other metamethods earn their keep on classes.

### `__tostring` — readable debug output
```lua
function Counter.__tostring(self): string
    return string.format("Counter(n=%d, step=%d)", self.n, self.step)
end

print(c)   -- Counter(n=15, step=5)   instead of   table: 0x...
```
Cheap to add, transforms `print` debugging from "where did this object come from?" to "obvious what this is."

### Typo-guarding enum-like tables

Plain table "enums" silently return `nil` for typos:
```lua
local Color = {Red = "Red", Green = "Green", Blue = "Blue"}
print(Color.Rdd)   -- nil — no error, you'll find out the hard way
```

Add an `__index` metamethod that errors on unknown keys:
```lua
local Color = setmetatable(
    {Red = "Red", Green = "Green", Blue = "Blue"},
    {__index = function(_, key)
        error(string.format("%q is not a valid Color", tostring(key)), 2)
    end}
)

print(Color.Rdd)   -- ERROR: "Rdd" is not a valid Color
```
The `, 2` argument to `error` blames the caller rather than this metamethod, so the stack trace points at where the bad access happened. Handy for any "constants table" you don't want fat-fingered. Pair with `table.freeze` if you also want to block writes.

## OOP-adjacent: ECS, services, signals

Pure OOP isn't the only option. For data-heavy systems, an Entity-Component-System (ECS) layout — entities are ids, components are data tables, systems are functions over filtered components — fits Roblox better than deep class hierarchies. Libraries like **Matter** implement this.

For singletons (managers, registries, services), don't use a class — return a single table from the module:
```lua
local Inventory = {}
Inventory._items = {}

function Inventory.Add(itemId: number) end
function Inventory.Get(itemId: number) end

return Inventory
```
That's a "service module," not a class. Don't `Inventory.new()` for it; just `require()`.

## State machines

A common shape for NPCs, doors, UI flows, and gameplay round logic: a small state machine. There's no Roblox-specific framework; the building blocks above are enough. Idiomatic Luau:

```lua
local NPC = {}
NPC.__index = NPC

type State = "Idle" | "Patrol" | "Chase" | "Attack" | "Dead"

function NPC.new(model: Model)
    local self = setmetatable({
        model = model,
        state = "Idle" :: State,
        target = nil :: Player?,
    }, NPC)
    return self
end

local handlers: {[State]: (NPC, dt: number) -> ()} = {
    Idle = function(self, dt) ... end,
    Patrol = function(self, dt) ... end,
    Chase = function(self, dt) ... end,
    Attack = function(self, dt) ... end,
    Dead = function(self, dt) end,
}

function NPC:Tick(dt: number)
    handlers[self.state](self, dt)
end

function NPC:Transition(to: State)
    -- exit current, enter new (place per-state setup/teardown here)
    self.state = to
end

return NPC
```

Pair with a `Heartbeat` driver or a per-NPC `task.spawn` loop. For more elaborate FSMs (hierarchical, parallel), reach for a community library — but for almost all gameplay states, this 30-line pattern is enough.

## Pitfalls

- **Forgetting `Counter.__index = Counter`.** Method calls error with "attempt to call a nil value" because Lua can't find `Increment` on the instance.
- **Mixing dot and colon by accident.** `Counter:new(...)` (colon) passes `Counter` as `self` — likely not what you wanted. Constructors are `Counter.new(...)` (dot).
- **Sharing state across instances.** Putting mutable state on the class table (`Counter.shared = {}`) makes it shared by every instance. Put per-instance state in `self` inside `.new`.
- **Inheritance chains > 1 level deep.** Refactor to composition.
- **No `:Destroy()` on a class that holds connections.** Memory leak.
- **`function class:method()` outside the module.** Adds the method on the consumer side, not the class. Always define methods inside the module that owns the class.
- **`self` collision when calling a method as a function.** `c.Increment()` (no colon) errors because `self` is missing. Always use `:` to call a method.
- **Using OOP for everything.** Many problems are simpler as a free function over a data table. Reach for a class when you have state + behaviour bundled together.

## See also
- [scripts-and-modules.md](scripts-and-modules.md) — module structure and lifecycle
- [luau-types.md](luau-types.md) — typing class instances with `typeof(setmetatable(...))`
- [luau-style.md](luau-style.md) — naming conventions for classes and members
- [luau-basics.md](luau-basics.md) — closures, upvalues, weak tables (for instance lifetimes)
- [round-based.md](round-based.md), [npcs.md](npcs.md), [status-effects.md](status-effects.md) — applied state-machine and class examples
- [community-libraries.md](community-libraries.md) — **Trove / Janitor** (RbxUtil) for one-shot cleanup of many connections
