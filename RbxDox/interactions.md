# Interactions (ProximityPrompt & ClickDetector)

> Upstream: <https://create.roblox.com/docs/ui/proximity-prompts> · <https://create.roblox.com/docs/reference/engine/classes/ProximityPrompt> · <https://create.roblox.com/docs/reference/engine/classes/ClickDetector>
> Source: distilled from the ProximityPrompt and ClickDetector API references

## Purpose
Two engine-provided primitives let players interact with world objects without you building UI: **`ProximityPrompt`** (modern, hold-to-activate, gamepad/touch-friendly, customizable) and **`ClickDetector`** (older, simpler click-only). Use a ProximityPrompt for nearly every new "press E to use this" interaction. Reach for ClickDetector only for legacy compatibility or when you specifically want a mouse-click feel without proximity restriction.

## Decision: which to use

| Need | Use |
|---|---|
| Door, vendor, button, lever, pickup, NPC dialog | **ProximityPrompt** |
| "Click anywhere on this big object" with no distance limit | ClickDetector with high `MaxActivationDistance` |
| Cross-platform (gamepad, touch, KB+M) | ProximityPrompt — touch and gamepad mappings are automatic |
| Hold-to-activate (defuse, open safe) | ProximityPrompt with `HoldDuration > 0` |
| Multiple alternative actions on one object | Multiple ProximityPrompts with different `KeyboardKeyCode` values |
| Mouse-hover feedback | ClickDetector (`MouseHoverEnter` / `Leave`) |

## ProximityPrompt

Parent under a **`BasePart`** or **`Attachment`**. The prompt UI appears when a player gets within `MaxActivationDistance` and faces the object (when `RequiresLineOfSight = true`).

### Key API
```lua
-- Properties
prompt.ActionText           -- "Open"  — verb shown
prompt.ObjectText           -- "Door"  — what's being acted on
prompt.KeyboardKeyCode      -- Enum.KeyCode.E  (default)
prompt.GamepadKeyCode       -- Enum.KeyCode.ButtonX (default)
prompt.HoldDuration         -- seconds; 0 = instant tap
prompt.MaxActivationDistance -- studs; default 10 (often too small)
prompt.RequiresLineOfSight  -- default true
prompt.Enabled              -- toggle on/off
prompt.Style                -- Default | Custom (Custom hides built-in UI; you draw your own)
prompt.UIOffset             -- Vector2 pixel nudge for the built-in UI
prompt.Exclusivity          -- OnePerButton | OneGlobally | AlwaysShow
prompt.AutoLocalize, prompt.ClickablePrompt

-- Events
prompt.Triggered:Connect(function(player) end)         -- successful activation
prompt.TriggerEnded:Connect(function(player) end)      -- end of hold
prompt.PromptShown:Connect(function(inputType) end)    -- player entered range / faced it
prompt.PromptHidden:Connect(function() end)
prompt.PromptButtonHoldBegan:Connect(function(player) end)
prompt.PromptButtonHoldEnded:Connect(function(player) end)
```

### Patterns

**Door — server-side state change:**
```lua
-- Server Script under the door part
local prompt = script.Parent:WaitForChild("Prompt")
local door = script.Parent
local TweenService = game:GetService("TweenService")

local isOpen = false
local closedCFrame = door.CFrame
local openCFrame = closedCFrame * CFrame.Angles(0, math.rad(90), 0)

prompt.Triggered:Connect(function(player)
    isOpen = not isOpen
    prompt.ActionText = if isOpen then "Close" else "Open"
    TweenService:Create(door, TweenInfo.new(0.4), {CFrame = if isOpen then openCFrame else closedCFrame}):Play()
end)
```

**Pickup with hold-to-confirm:**
```lua
prompt.HoldDuration = 1
prompt.ActionText = "Take"
prompt.Triggered:Connect(function(player)
    -- grant item server-side
    grantItem(player, "HealthPotion")
    script.Parent:Destroy()   -- remove the world pickup
end)
```

**Multiple actions on one object:**
```lua
-- Parent both prompts to the same Part
local examine = createPrompt({KeyboardKeyCode = Enum.KeyCode.E, ActionText = "Examine"})
local take = createPrompt({KeyboardKeyCode = Enum.KeyCode.F, ActionText = "Take"})

examine.Triggered:Connect(function(player) showLore(player) end)
take.Triggered:Connect(function(player) grant(player); script.Parent:Destroy() end)
```

**Custom UI (Style = Custom):**
```lua
prompt.Style = Enum.ProximityPromptStyle.Custom
local ProximityPromptService = game:GetService("ProximityPromptService")

ProximityPromptService.PromptShown:Connect(function(p, inputType)
    if p ~= prompt then return end
    -- Draw your own BillboardGui adjacent to the prompt's parent
end)
```
The Custom style suppresses the default UI; you receive `PromptShown`/`Hidden`/`Triggered` and render whatever you want.

### Pitfalls

- **`Triggered` fires on whichever side has the script.** A `Script` inside the same Part fires server-side; a `LocalScript` (in PlayerScripts, listening via `ProximityPromptService.PromptTriggered`) fires client-side. **Server-side is the right place for state changes.**
- **`MaxActivationDistance = 10` (default) is small.** For obvious interactables, 15–25 is more usable.
- **`RequiresLineOfSight = true` and prompt parented inside the part** — the part itself blocks the line of sight check. Either set false or parent under an Attachment offset away from the geometry.
- **Hold prompts don't auto-cancel on input release** if the player moves out of range mid-hold. Use `PromptButtonHoldEnded` if you need to react to that.
- **Same `KeyboardKeyCode` on overlapping prompts** — engine picks one based on `Exclusivity`. Use different keys or set Exclusivity explicitly.
- **`Triggered` validation.** The `player` arg is trusted (server-routed), but they could be at any distance if you fire from the client side — always re-check distance on the server if it matters.

## ClickDetector

Parent under a **`BasePart`** or **`Model`** (uses the model's bounding box).

### Key API
```lua
detector.MaxActivationDistance     -- 32 studs default; 0 = unlimited (within visible range)
detector.CursorIcon                -- asset id for hover cursor
detector.MouseHoverEnabled         -- bool

detector.MouseClick:Connect(function(player) end)
detector.RightMouseClick:Connect(function(player) end)
detector.MouseHoverEnter:Connect(function(player) end)
detector.MouseHoverLeave:Connect(function(player) end)
```

### Pattern — clickable button
```lua
-- Server Script under the button part
local detector = script.Parent:WaitForChild("ClickDetector")
detector.MouseClick:Connect(function(player)
    -- toggle, grant, open, etc.
end)
```

### Pitfalls

- **No keyboard / gamepad equivalent.** Touch works (counts as a tap), but a controller-only player can't activate a ClickDetector unless your code routes to it via ContextActionService.
- **Mouse-only hover events** — touch and gamepad don't fire MouseHoverEnter/Leave.
- **Distance check is client-side reported but server-trusted** — the server still receives MouseClick with the correct player, but the player may be slightly out of range due to lag.
- **No `Enabled` toggle for granular control** — set `MaxActivationDistance = 0` or `:Destroy()` the detector to disable.

## Choosing — practical rules

- New code: **default to ProximityPrompt.** Cross-platform, accessible, customizable, hold-to-act.
- Migrating legacy code: leave ClickDetectors in place until they need behaviour ProximityPrompt would do better.
- "Big clickable object" (sign, billboard, easter egg) where you don't want a hovering prompt UI: ClickDetector is fine.

## See also
- [events-signals.md](events-signals.md) — server-side handlers for Triggered/MouseClick are RBXScriptSignals
- [security.md](security.md) — `Triggered`/`MouseClick` provide the `player`, but always validate ownership/range/cooldown server-side
- [ui-system.md](ui-system.md) — for in-world UI that doesn't fit either primitive
- [tweens-animation.md](tweens-animation.md) — common to tween the affected object on Triggered
- [tools.md](tools.md) — for items that the player carries; Tools and Interactions cover different surfaces
- [world-mechanics.md](world-mechanics.md) — passive world-mechanic alternatives (touch-pads, conveyors, jump pads) that don't need a prompt
- [vehicles.md](vehicles.md) — Seat / VehicleSeat for sit-to-use interactions
- [npcs.md](npcs.md) — vendor / quest-giver / dialog NPCs use ProximityPrompt for activation
- [shops.md](shops.md), [quests.md](quests.md) — common destinations for `Triggered`
