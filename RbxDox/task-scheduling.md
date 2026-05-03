# Task Scheduling

> Upstream: <https://create.roblox.com/docs/scripting/scheduler> · <https://create.roblox.com/docs/scripting/multithreading>
> Source: `content/en-us/scripting/scheduler.md`, `content/en-us/scripting/multithreading.md`

## Purpose
Roblox runs Luau on a cooperative scheduler — your code only yields when you tell it to. The `task` library is the modern API for sleeping, deferring, and spawning. **Always use `task.*` over the legacy `wait`/`spawn`/`delay`** — the old ones are slower, throttled, and deprecated.

## The task library

```lua
task.wait(t?)        -- yield current thread for at least t seconds (default 0). Returns elapsed.
task.spawn(f, ...)   -- run f IMMEDIATELY in a new thread, then return to caller
task.defer(f, ...)   -- schedule f to run at the next resumption point (after current resumption finishes)
task.delay(t, f, ...) -- run f after t seconds in a new thread
task.cancel(thread)  -- kill a thread
task.synchronize()   -- jump to serial execution (parallel Luau)
task.desynchronize() -- jump to parallel execution
```

## spawn vs defer
- `task.spawn(f)` runs `f` right now (and the calling code resumes when `f` yields or returns).
- `task.defer(f)` queues `f` for the *end* of the current resumption cycle — after every currently-running thread reaches a yield. Useful for "run after the current event handler finishes" without nesting.

## RunService for frame-locked work
For per-frame logic, prefer `RunService` signals over a `while task.wait()` loop:

```lua
local RunService = game:GetService("RunService")

-- Client only — runs every frame, before render
RunService.RenderStepped:Connect(function(dt)
    -- camera, UI, particle position
end)

-- Server or client — runs every physics step
RunService.Heartbeat:Connect(function(dt)
    -- gameplay tick
end)
```

`while task.wait() do` runs roughly every 1/60s but you have no control over phasing relative to physics or render. Use the right signal.

## Patterns

### Cooldown
```lua
local lastFire = 0
local COOLDOWN = 0.5

local function tryFire()
    if os.clock() - lastFire < COOLDOWN then return end
    lastFire = os.clock()
    -- fire
end
```

### Debounce per object
```lua
local debounced = {}
part.Touched:Connect(function(hit)
    local hum = hit.Parent and hit.Parent:FindFirstChildOfClass("Humanoid")
    if not hum or debounced[hum] then return end
    debounced[hum] = true
    hum:TakeDamage(10)
    task.delay(1, function() debounced[hum] = nil end)
end)
```

### Background work
```lua
task.spawn(function()
    while true do
        task.wait(60)
        autosave()
    end
end)
```

### Fire-and-forget remote handler
Some `OnServerEvent` handlers should not block the network thread. Wrap heavy work:
```lua
remote.OnServerEvent:Connect(function(player, ...)
    task.spawn(function()
        doExpensiveWork(player, ...)
    end)
end)
```

## Parallel Luau

For CPU-bound work (pathfinding bulk queries, procedural generation, voxel updates), use **Actors**:

1. Create an `Actor` Instance.
2. Put a `Script` (with `RunContext = Server` or any) inside it.
3. From inside the actor's script, `task.desynchronize()` to enter parallel mode; `task.synchronize()` to come back to the main thread before touching shared state.

Within parallel mode you can read most properties but cannot write to the shared DataModel. Use `SharedTable` (mutex-free concurrent table) for cross-actor data. Use `Actor:SendMessage(topic, ...)` and `Actor:BindToMessage(topic, fn)` for actor-to-actor messaging.

Don't reach for parallel until you've profiled and the serial path is the bottleneck.

## Pitfalls

- **`wait()`** and **`spawn()`** without `task.` are deprecated, throttled to a minimum of ~30 ms, and slower. Always prefix with `task.`.
- **Tight `while task.wait(0) do` loops** still hand control back, but you'll burn a frame every iteration. Use `RunService.Heartbeat` if you genuinely need every step.
- **Long-running yields inside event handlers** with `Deferred` signal behaviour can subtly reorder things. If correctness depends on order, fire one event and let the handler chain explicitly.
- **Forgetting `task.cancel`.** A thread that's `task.delay`d and then no longer needed will still run. Cancel it on cleanup.
- **Errors in `task.spawn`.** Errors in spawned threads print to Output but don't propagate. Wrap in `xpcall` if you need to handle them.

## See also
- [events-signals.md](events-signals.md) — deferred behaviour
- [services.md](services.md) — RunService overview
- [performance.md](performance.md) — when to reach for parallel Luau
