# UI System

> Upstream: <https://create.roblox.com/docs/ui>
> Source: `content/en-us/ui/index.md`, `frames.md`, `position-and-size.md`, `size-modifiers.md`, `list-flex-layouts.md`

## Purpose
Roblox UI is built from `GuiObject` Instances inside a container — usually `ScreenGui` (for player HUD/menus). Layout is done with `UDim2` (combined scale + offset) and automatic-layout helpers (`UIListLayout`, `UIGridLayout`, `UIFlexItem`). It's its own little subsystem; once you internalize UDim2 and anchor points, the rest is straightforward.

## Containers

| Container | Where it shows |
|---|---|
| `ScreenGui` | 2D overlay on the player's screen (parent under `PlayerGui`, or place under `StarterGui` to auto-copy on spawn) |
| `BillboardGui` | World-space, always faces the camera; parent under a part |
| `SurfaceGui` | World-space, painted on a Part's face |

A `ScreenGui` lives under `PlayerGui` (per-player). To make UI for everyone, put it in `StarterGui` — it's copied to each player's PlayerGui on spawn (controlled by `ResetOnSpawn`).

## Common GuiObjects

- **`Frame`** — empty container; the building block.
- **`TextLabel`** — text, no input.
- **`TextButton`** — clickable text.
- **`ImageLabel`** — image, no input.
- **`ImageButton`** — clickable image.
- **`TextBox`** — editable text input.
- **`ScrollingFrame`** — frame with scroll bars (set `CanvasSize`, or pair with `UIListLayout` + `AutomaticCanvasSize`).
- **`ViewportFrame`** — renders 3D content in 2D UI.
- **`VideoFrame`** — plays a video asset.

All inherit common properties: `Position`, `Size`, `AnchorPoint`, `BackgroundColor3`, `BackgroundTransparency`, `BorderSizePixel`, `Visible`, `ZIndex`, `Rotation`, `LayoutOrder`.

## UDim2 — position and size

`UDim2.new(scaleX, offsetX, scaleY, offsetY)` or `UDim2.fromScale(sX, sY)` or `UDim2.fromOffset(oX, oY)`.

- **Scale** is fraction of parent (0 = none, 1 = full parent dimension). Resolution-independent.
- **Offset** is absolute pixels. Resolution-dependent.

```lua
frame.Position = UDim2.fromScale(0.5, 0.5)         -- center of parent
frame.AnchorPoint = Vector2.new(0.5, 0.5)          -- pivot at frame's own center
frame.Size = UDim2.new(0.4, 0, 0.3, 0)             -- 40% wide, 30% tall

frame.Size = UDim2.new(0, 200, 0, 100)             -- 200×100 px (avoid for cross-platform)
frame.Size = UDim2.new(0.4, 0, 0, 100)             -- 40% wide, 100 px tall (mixed)
```

### AnchorPoint
The fraction-of-self that aligns to `Position`. `Vector2.new(0.5, 0.5)` makes Position the center; `Vector2.new(1, 0)` makes Position the top-right corner.

## Layouts (auto-arrange children)

Drop one of these as a child of a container; it lays out the siblings.

- **`UIListLayout`** — vertical or horizontal list, with padding and alignment.
- **`UIGridLayout`** — fixed-cell grid.
- **`UITableLayout`** — rows-as-children, cells-within-rows.
- **`UIPageLayout`** — paginated, animated transitions.
- **`UIAspectRatioConstraint`** — keeps a child at a target aspect ratio.
- **`UISizeConstraint`** / **`UITextSizeConstraint`** — clamp pixel sizes.
- **`UIPadding`** — inner padding.
- **`UICorner`** — rounded corners (set `CornerRadius`).
- **`UIStroke`** — outline.
- **`UIGradient`** — color/transparency gradient.
- **`UIScale`** — multiply child sizes (good for responsive scaling).
- **`UIFlexItem`** — flexbox-style growth/shrink (modern).

```lua
local list = Instance.new("UIListLayout", parent)
list.FillDirection = Enum.FillDirection.Vertical
list.HorizontalAlignment = Enum.HorizontalAlignment.Center
list.Padding = UDim.new(0, 8)
list.SortOrder = Enum.SortOrder.LayoutOrder
```
Children render in `LayoutOrder` (or `Name` if `SortOrder = Name`).

## Automatic sizing

`AutomaticSize` lets a container size itself to fit children:
```lua
frame.AutomaticSize = Enum.AutomaticSize.Y    -- height grows with content
frame.AutomaticSize = Enum.AutomaticSize.XY
```
Pair with `UIListLayout` and `UIPadding` for self-sizing menus.

For ScrollingFrame, set `AutomaticCanvasSize = Enum.AutomaticSize.Y` so the canvas grows.

## Patterns

### Centered modal
```lua
local screen = Instance.new("ScreenGui")
screen.IgnoreGuiInset = true
screen.Parent = playerGui

local modal = Instance.new("Frame", screen)
modal.AnchorPoint = Vector2.new(0.5, 0.5)
modal.Position = UDim2.fromScale(0.5, 0.5)
modal.Size = UDim2.fromOffset(400, 300)
modal.BackgroundColor3 = Color3.fromRGB(40, 40, 40)

Instance.new("UICorner", modal).CornerRadius = UDim.new(0, 12)
local pad = Instance.new("UIPadding", modal)
pad.PaddingTop = UDim.new(0, 16); pad.PaddingBottom = UDim.new(0, 16)
pad.PaddingLeft = UDim.new(0, 16); pad.PaddingRight = UDim.new(0, 16)
```

### Button click
```lua
button.MouseButton1Click:Connect(function()
    print("clicked")
end)

-- Or for any input device:
button.Activated:Connect(function() ... end)
```

### Hover state via signals
```lua
button.MouseEnter:Connect(function() button.BackgroundTransparency = 0.2 end)
button.MouseLeave:Connect(function() button.BackgroundTransparency = 0.5 end)
```

### Responsive scrolling list
```lua
local scroll = Instance.new("ScrollingFrame", parent)
scroll.Size = UDim2.fromScale(1, 1)
scroll.CanvasSize = UDim2.new()                       -- 0,0 — auto will fill
scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
scroll.ScrollBarThickness = 6

local list = Instance.new("UIListLayout", scroll)
list.SortOrder = Enum.SortOrder.LayoutOrder
list.Padding = UDim.new(0, 4)
```

## Pitfalls

- **All-pixel sizes (`Offset` only)** look great on your monitor and tiny on a phone. Use `Scale` for responsiveness, `Offset` only for fine padding.
- **Forgetting `AnchorPoint`** when centering — `UDim2.fromScale(0.5, 0.5)` puts the *top-left* of your frame at the screen center.
- **`UIListLayout` overrides children's positions.** Don't set Position on layout-managed siblings.
- **`MouseButton1Click`** doesn't fire on touch. `Activated` does, on all input types.
- **Nested ZIndex** — by default, child ZIndex is local; set `ScreenGui.ZIndexBehavior = Global` to make ZIndex absolute across the tree.
- **UI on the server.** Doesn't render. Build UI from LocalScripts.
- **Forgetting `IgnoreGuiInset`.** The top bar (`PlayerGui` default inset) crops your UI. Set `IgnoreGuiInset = true` for full-screen UI.
- **`ResetOnSpawn`** copies StarterGui to PlayerGui every respawn — your live UI gets wiped. Set `false` if you build UI dynamically.

## See also
- [menu-systems.md](menu-systems.md) — composing primitives into menus (state manager, tabs, transitions, dialogs, loading screens)
- [tweens-animation.md](tweens-animation.md) — animating UI properties
- [input-handling.md](input-handling.md) — gameProcessed and UI input
- [shops.md](shops.md), [quests.md](quests.md), [save-slots.md](save-slots.md) — common consumers of the UI primitives covered here
- [community-libraries.md](community-libraries.md) — **Roact / React-Lua** for declarative UI; **TopbarPlus** for menus
