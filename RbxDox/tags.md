# Tags & CollectionService

> Upstream: <https://create.roblox.com/docs/reference/engine/classes/CollectionService>
> Source: distilled from the CollectionService API and the tag-driven architecture pattern used widely in Roblox open-source projects

## Purpose
A **tag** is a string label you attach to any Instance. **`CollectionService`** lets you query "all Instances with this tag," subscribe to tag-add and tag-remove events, and structure your code around behaviour buckets rather than hard-coded paths or class hierarchies. This is one of the most important architectural patterns in Roblox — it lets a single script handle every "Hazard" or "Pickup" or "Door" in the world without knowing where they're parented or what they're named.

## Key API

```lua
local CollectionService = game:GetService("CollectionService")

-- Mutate
instance:AddTag("Hazard")                          -- Instance method (modern; sugar for CollectionService:AddTag)
instance:RemoveTag("Hazard")
instance:HasTag("Hazard")                          -- → bool
instance:GetTags()                                 -- → {string}

CollectionService:AddTag(instance, "Hazard")       -- equivalent service-style calls
CollectionService:RemoveTag(instance, "Hazard")
CollectionService:HasTag(instance, "Hazard")
CollectionService:GetTags(instance)

-- Query
CollectionService:GetTagged("Hazard")              -- → array of all currently-tagged Instances
CollectionService:GetAllTags()                     -- → array of all tag strings used in the place

-- React
CollectionService:GetInstanceAddedSignal("Hazard"):Connect(function(instance) end)
CollectionService:GetInstanceRemovedSignal("Hazard"):Connect(function(instance) end)
```

Tags **replicate** from server to client (including those set in Studio at edit time). The Studio UI for adding tags is **View → Tag Editor** (built-in).

## The setup-on-add pattern (the most important pattern in this card)

The canonical way to use tags: one script owns a behaviour, finds all currently-tagged Instances, sets each up, and listens for newly-tagged ones.

```lua
local CollectionService = game:GetService("CollectionService")

local function setupHazard(hazard: BasePart)
    local damage = hazard:GetAttribute("Damage") or 25
    local debounce = {}

    hazard.Touched:Connect(function(hit)
        local hum = hit.Parent and hit.Parent:FindFirstChildOfClass("Humanoid")
        if not hum or debounce[hum] then return end
        debounce[hum] = true
        hum:TakeDamage(damage)
        task.delay(1, function() debounce[hum] = nil end)
    end)
end

-- Set up everything currently tagged
for _, hazard in CollectionService:GetTagged("Hazard") do
    setupHazard(hazard)
end

-- Set up anything added later (server-spawned, client-streamed-in, designer-tagged at runtime)
CollectionService:GetInstanceAddedSignal("Hazard"):Connect(setupHazard)
```

This is a small idiom but it scales: one script handles all hazards, regardless of where they live in the tree, what they're named, or how they got there. Designers tag parts in Studio without touching code; the runtime picks them up automatically.

For cleanup symmetry:
```lua
CollectionService:GetInstanceRemovedSignal("Hazard"):Connect(function(hazard)
    -- e.g., disconnect anything you connected, kill any threads, release pooled resources
end)
```

## Tags vs Attributes — when to use which

They are complementary, not competing:

| | **Tag** | **Attribute** |
|---|---|---|
| What it expresses | "this is a Hazard" (kind / behavior bucket) | "Damage = 25" (data on this Instance) |
| Type | String label | Typed value (number, string, Vector3, etc.) |
| Per-Instance | Many tags allowed | Many attributes allowed |
| Queryable | Yes — `CollectionService:GetTagged` | No — must walk the tree |
| Reactive | `GetInstance{Added,Removed}Signal(tag)` | `GetAttributeChangedSignal(name)` |

**Use both together:** tag a Part as `"Hazard"` to identify its kind; use Attributes (`Damage`, `Cooldown`, `KnockbackDir`) for its tunable parameters. See [attributes.md](attributes.md).

```lua
-- Designer's Studio workflow:
-- 1. Select a Part. Open Tag Editor. Add tag "Hazard".
-- 2. Open Properties → Attributes. Add Damage = 30, Cooldown = 0.5.
-- 3. Done. No script changes; the Hazard system handles it on next play.
```

## Patterns

### Marking NPC types
```lua
-- Tag every spawned enemy with its archetype
zombie:AddTag("Enemy")
zombie:AddTag("Zombie")

-- Other systems can react to broad or specific groups
for _, e in CollectionService:GetTagged("Enemy") do dropAggro(e) end
for _, z in CollectionService:GetTagged("Zombie") do z:GetAttribute("InfectionRate") end
```

### Cross-script wiring (avoid hard requires)
A door system tags an interactable. A separate sound system listens for the same tag and plays a sound when triggered. Neither script needs to know about the other beyond the tag string — loose coupling.

### Lifetime-bound tag
```lua
-- Tag a part for "this round only," remove on round end
for _, p in CollectionService:GetTagged("RoundPickup") do p:RemoveTag("RoundPickup") end
-- The corresponding GetInstanceRemovedSignal handlers run cleanup.
```

### Pre-cache for hot paths
```lua
-- Instead of CollectionService:GetTagged(...) every frame, cache and update on signals
local hazards: {[BasePart]: true} = {}
for _, h in CollectionService:GetTagged("Hazard") do hazards[h] = true end
CollectionService:GetInstanceAddedSignal("Hazard"):Connect(function(h) hazards[h] = true end)
CollectionService:GetInstanceRemovedSignal("Hazard"):Connect(function(h) hazards[h] = nil end)
-- Now iterate `hazards` per-frame; O(n) without the service call overhead.
```
See [performance.md](performance.md) for the broader "cache descendants" pattern.

## Pitfalls

- **Tags are strings — typos silently match nothing.** No autocomplete, no compiler check. Use a constants module for tag names if you reference them across many scripts:
  ```lua
  -- ReplicatedStorage/Tags.lua
  return table.freeze({
      Hazard = "Hazard",
      Pickup = "Pickup",
      Enemy = "Enemy",
  })
  ```
  Then `instance:AddTag(Tags.Hazard)` — typo becomes a Luau error instead of a silent miss.
- **`GetTagged` returns a snapshot.** Don't iterate it expecting to see Instances tagged later in the same frame. Use the `GetInstanceAddedSignal` for live updates.
- **Removed-signal does NOT fire on `:Destroy()`.** When an Instance is destroyed, its tags vanish but `GetInstanceRemovedSignal` doesn't fire. If you need destroy-time cleanup, also connect `instance.Destroying`.
- **Tags don't show in the Properties panel by default.** Use the Tag Editor (View → Tag Editor) to see/edit them. They're visible in the Tags property if you scroll to it.
- **Tagging in edit mode requires the Tag Editor.** A `:AddTag` call from a Script during play-test only persists for the session; for permanent tags, set them in Studio's Tag Editor or via a plugin.
- **Per-frame `GetTagged` in a hot loop** — every call walks the internal map. Cache as shown in Patterns above for hot paths.
- **Mass-tagging Instances on the client** — the client can tag locally, but those tags don't replicate back to the server. For shared state, tag server-side.
- **Confusing tags with CollectionService's older "set" semantics.** Modern API is purely tag-based; ignore old DevForum posts using the deprecated single-set behaviour.

## See also
- [attributes.md](attributes.md) — the natural partner to tags for typed per-Instance config
- [services.md](services.md) — CollectionService overview
- [project-structure.md](project-structure.md) — tags decouple behaviour from tree placement
- [performance.md](performance.md) — caching tagged Instances for hot loops
- [interactions.md](interactions.md) — common to tag interactable objects then attach a ProximityPrompt by tag
