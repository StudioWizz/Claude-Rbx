# Performance & Optimization

> Upstream: <https://create.roblox.com/docs/performance-optimization>
> Source: `content/en-us/performance-optimization/index.md`, `improve.md`, `microprofiler/index.md`, `content/en-us/workspace/streaming/index.md`

## Purpose
Roblox runs on everything from a $1500 PC to a 4-year-old phone. Performance discipline is the difference between an experience that works for 10% of your audience and 100%. The cycle: **identify** (profile) → **design** (architectural fixes) → **improve** (targeted optimization) → **monitor** (catch regressions).

Don't optimize without profiling. The bottleneck is rarely where you guess.

## MicroProfiler

Roblox's built-in frame profiler. Open with **Ctrl/Cmd+F6** in Studio, or `Settings → MicroProfiler` in a published experience.

- Each row is a thread; each colored bar is a labeled timed section.
- Hover for self / total time per frame.
- `Ctrl+P` to pause; click bars to drill in.
- "Frame breakdown" view shows the % each system used.

You can add your own labels:
```lua
debug.profilebegin("UpdateAI")
-- expensive thing
debug.profileend()
```
Or with the convenience wrapper:
```lua
debug.profilebegin("Foo"); doFoo(); debug.profileend()
```

The `Network` and `Memory` views inside MicroProfiler show replication and memory usage — useful when frame time is fine but bandwidth or RAM is the issue.

## Common bottlenecks (in rough order of frequency)

1. **Too many parts.** Anchored geometry is cheap to render in bulk, but tens of thousands of unanchored parts simulate poorly. Use Streaming.
2. **Per-frame Instance creation.** `Instance.new` + `Parent` is moderately fast, but doing it dozens of times per Heartbeat hammers the engine. Pool. Related: when constructing many Instances at once, **build then parent** — assemble inside an unparented root and assign `Parent` once at the end, so the engine performs one replication step instead of N. See [workspace-hierarchy.md](workspace-hierarchy.md) and [parts-and-models.md](parts-and-models.md).
3. **Touched storms.** `Touched` fires every step a contact exists. Always debounce.
4. **Unfiltered `GetDescendants`.** Walking thousands of descendants every frame. Cache, or use `CollectionService:GetTagged(tag)`.
5. **High-poly meshes with `Precise` collision.** CollisionFidelity dominates physics cost. Drop to `Hull` or `Box`.
6. **Big DataStore writes on hot paths.** Save on events, not in `Heartbeat`.
7. **Inefficient remotes.** Firing every frame, sending big tables. Throttle to 10–30 Hz; use UnreliableRemoteEvent for state broadcasts.
8. **Lots of small UDim2-driven UI elements** with `AutomaticSize` everywhere. Layout engines reflow per change.
9. **`wait()` / `spawn()`.** Deprecated, throttled. Always use `task.*`.
10. **Expensive raycasts every frame from many sources.** Batch, cache, or shape-cast once per Heartbeat instead of per-RenderStep.

## Streaming

Streaming streams in / out parts of `Workspace` based on player position. Set on Workspace:

```lua
workspace.StreamingEnabled = true
workspace.StreamingMinRadius = 64       -- always loaded around player
workspace.StreamingTargetRadius = 256   -- aim to load up to this
workspace.StreamOutBehavior = Enum.StreamOutBehavior.Default
workspace.StreamingIntegrity = Enum.StreamingIntegrity.Default
```

When enabled:
- The client may not have every Instance you expect — use `:WaitForChild` aggressively.
- Use `Model.ModelStreamingMode` to control whether a model streams as a unit.
- Server scripts always see everything; only the client view is partial.

Streaming makes large worlds (cities, sprawling RPG maps) feasible on lower-end hardware. Without it, every part is loaded for everyone, forever.

## Design patterns that scale

- **Tagging + CollectionService** for typed Instance handling instead of name-walking the tree.
- **Object pools** for projectiles, NPCs, particles. Reuse a small set; rotate inactive copies into a hidden parent.
- **Server-authoritative simulation, client-predicted visuals** for combat/movement. Snap to server state if drift exceeds threshold.
- **Spatial partitioning** (region-based or grid) for AI/effect updates instead of "tick every NPC every frame."
- **LOD (Level of Detail).** Switch a complex MeshPart for a simple block when distant; use SubjectDistance from CurrentCamera.
- **Throttle networking.** Compress repeated sends, send deltas, use UnreliableRemoteEvent for noisy data.
- **Deferred work.** Use `task.defer` to hand off non-urgent work to the next frame.

## Memory

Watch in the **Memory** tab of MicroProfiler or `View → Performance Stats → Memory`. Common leaks:

- Connections not disconnected (the closure pins the upvalues).
- ModuleScripts that hold growing tables forever (caches).
- Forgotten `Instance.new` results (parented to nil but referenced).
- Caches keyed by Lua tables/functions that you never clear — fix with `setmetatable(cache, {__mode = "k"})` so unreferenced keys auto-drop. **Does not work for Instance keys** — those need explicit cleanup via `Destroying` / your own teardown. See [luau-basics.md → Garbage collection and weak tables](luau-basics.md).

Use `Instance:Destroy()` to fully release. Setting `Parent = nil` alone leaves connections live.

## Network

Per-player bandwidth budget is small (~50 KB/s for replication is healthy). Big sources:
- Frequently-updated Instance properties on many parts.
- Large RemoteEvent payloads.
- Streaming new chunks (manage with Streaming radii).

Use **Stats Service** programmatically:
```lua
local Stats = game:GetService("Stats")
print(Stats.DataReceiveKbps, Stats.DataSendKbps)
```

## Parallel Luau (last resort)

For genuine CPU-bound work, run scripts inside `Actor` Instances with `task.desynchronize()`. See [task-scheduling.md](task-scheduling.md). Don't reach for it before profiling proves the serial path is the bottleneck — the synchronization rules are easy to get wrong.

## Patterns

### Profile a function
```lua
debug.profilebegin("PathfindBatch")
for _, npc in npcs do recompute(npc) end
debug.profileend()
```

### Cache descendants
```lua
local hazards = {}
for _, t in CollectionService:GetTagged("Hazard") do
    table.insert(hazards, t)
end
CollectionService:GetInstanceAddedSignal("Hazard"):Connect(function(t) table.insert(hazards, t) end)
-- now you iterate `hazards`, not the whole tree
```

### Hoist out of hot loops
Lua/Luau resolves table-indexed names every access. In tight per-frame loops, lift them into locals:

```lua
-- BAD — every iteration re-indexes Vector3.new and SETTINGS
for i = 1, 10000 do
    positions[i] = Vector3.new(SETTINGS.Spacing * i, 0, 0)
end

-- GOOD — single index per name
local newVec = Vector3.new
local spacing = SETTINGS.Spacing
for i = 1, 10000 do
    positions[i] = newVec(spacing * i, 0, 0)
end
```

Also applies to `math.*`, `table.*`, deep property paths, and module-level constants you read in a tight loop. **Profile first** — for loops under a few thousand iterations the gain is noise.

### Object pool for projectiles
```lua
local pool = {}
local function get()
    return table.remove(pool) or makeProjectile()
end
local function release(p)
    p.Parent = nil
    table.insert(pool, p)
end
```

## Pitfalls

- **Optimizing without profiling.** You'll spend hours speeding up code that isn't the bottleneck.
- **Treating Studio's perf as the truth.** Studio's overhead is unrepresentative; test on a real device or Team Test.
- **Streaming with hard-coded `WaitForChild` everywhere.** When chunks unload and reload, your handlers may already be torn down. Design with the lifecycle in mind.
- **Re-creating tweens and connections every frame.** Cache them.
- **Server tick + client tick mismatch.** Client at 144Hz, server at 60Hz physics. Don't expect frame-perfect sync.
- **Memory leaks from anonymous closures.** A connection captured in `RunService.Heartbeat:Connect(function() ... part.Position ... end)` keeps `part` alive forever even if Destroyed.

## See also
- [task-scheduling.md](task-scheduling.md) — parallel Luau and task scheduler
- [collisions.md](collisions.md) — CollisionFidelity, CanQuery
- [studio-environment.md](studio-environment.md) — opening MicroProfiler
- [vfx.md](vfx.md), [lighting.md](lighting.md) — particle counts and post-effect costs
- [tags.md](tags.md) — caching tagged Instances for hot loops
- [workspace-hierarchy.md](workspace-hierarchy.md) — build-then-parent and Streaming setup
