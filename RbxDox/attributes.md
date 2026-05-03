# Attributes

> Upstream: <https://create.roblox.com/docs/scripting/attributes>
> Source: `content/en-us/scripting/attributes.md`

## Purpose
Attributes are typed key/value pairs you attach to any Instance via the Properties panel or the API. They're the right home for designer-tunable values (damage, cooldown, color) — better than hard-coded constants in scripts and better than CollectionService tags for anything more than a label.

## Supported types
`string`, `boolean`, `number`, `UDim`, `UDim2`, `BrickColor`, `Color3`, `Vector2`, `Vector3`, `CFrame`, `NumberSequence`, `ColorSequence`, `NumberRange`, `Rect`, `Font`, `EnumItem` (Luau), `Instance` (since recent updates).

Names must start with a letter and contain only letters, digits, and underscores. The `RBX*` prefix is reserved.

## API
```lua
instance:SetAttribute("Damage", 25)
local d = instance:GetAttribute("Damage")
local all = instance:GetAttributes()  -- {Damage = 25, ...}
instance:SetAttribute("Damage", nil)  -- delete

-- React to changes
instance:GetAttributeChangedSignal("Damage"):Connect(function()
    print("Damage now", instance:GetAttribute("Damage"))
end)

instance.AttributeChanged:Connect(function(name)
    -- fires for any attribute change
end)
```

## Patterns

### Designer-friendly tunables on a part
1. In Studio, select the part → Properties → Attributes (`+`) → add `Damage` (number) = `25`, `Cooldown` (number) = `1`, `KnockbackDir` (Vector3) = `0,1,0`.
2. In script:
```lua
local sword: Part = script.Parent
local lastHit = 0

sword.Touched:Connect(function(hit)
    local now = os.clock()
    if now - lastHit < sword:GetAttribute("Cooldown") then return end
    lastHit = now

    local hum = hit.Parent and hit.Parent:FindFirstChildOfClass("Humanoid")
    if hum then
        hum:TakeDamage(sword:GetAttribute("Damage"))
    end
end)
```
The designer can now retune damage and cooldown without touching code.

### Replication
Attributes replicate just like properties — set on the server, the client sees the value. Set on the client, only that client sees it.

### Pair with CollectionService
Use a tag to mark "this is a sword," then read attributes for its stats:
```lua
local CollectionService = game:GetService("CollectionService")
for _, sword in CollectionService:GetTagged("Sword") do
    local dmg = sword:GetAttribute("Damage") or 10
    -- ...
end
```

## Pitfalls

- **Forgetting attributes don't have a default.** `GetAttribute` returns `nil` if unset — always have a fallback.
- **Storing tables.** Attributes are scalars; tables aren't supported. Use a child `StringValue` with JSON, or a ModuleScript with config, or split into multiple attributes.
- **Per-frame `GetAttribute` polling.** Connect to `GetAttributeChangedSignal` instead.
- **Putting secret values on a replicated Instance.** Anything in `Workspace`/`ReplicatedStorage` ships to clients, including its attributes. Server-only data stays in `ServerStorage`.

## See also
- [services.md](services.md) — CollectionService for tags
- [project-structure.md](project-structure.md) — what replicates
