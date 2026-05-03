# Menu Systems

> Source: distilled from menu architecture patterns across genres (title screen, pause menu, settings, tabbed UIs, modal stacks)

## Purpose
`ui-system.md` covers the **primitives** (ScreenGui, Frame, UDim2, layouts). This card covers the **composition layer**: how to organize a game's menus as a coherent system. A typical game has a title/main menu, an in-game pause menu, a settings menu with tabs, several gameplay menus (inventory, shop, quest log), confirmation dialogs, tooltips, and a loading screen — plus the rules for what's open when, what hotkey toggles which menu, and what happens when you press ESC inside a sub-menu. Without an explicit menu-state manager, these grow into a tangle of conflicting visibility flags and broken-back-button bugs.

## Architecture

```
PlayerGui (per-player; created on join from StarterGui)
├── HUD (ScreenGui — always visible during gameplay)
│   ├── HealthBar
│   ├── AmmoCounter
│   └── Toasts (notification stack — see live-ops.md)
├── Menus (ScreenGui — holds all toggleable menus)
│   ├── MainMenu      (visible at game start)
│   ├── PauseMenu     (toggled by ESC)
│   ├── SettingsMenu  (sub-menu of pause)
│   ├── InventoryMenu (toggled by I)
│   ├── ShopMenu      (opened by vendor NPC)
│   └── ConfirmDialog (modal; appears over everything)
└── (Other ScreenGuis as needed)
```

Set `ScreenGui.ResetOnSpawn = false` for any menu/HUD that should survive respawn (which is most of them — only the round-result banner type things should reset).

## Menu state manager — single canonical pattern

The single most-impactful pattern: **only one major menu open at a time**, plus a **modal stack** for dialogs and tooltips that stack over everything.

```lua
--!strict
local MenuManager = {}

local current: string? = nil               -- name of the currently-open major menu
local stack: {string} = {}                  -- modal dialogs/tooltips on top
local menus: {[string]: ScreenGui} = {}     -- registered menus

local listeners: {[string]: {(open: boolean) -> ()}} = {}    -- per-menu open/close listeners

function MenuManager.Register(name: string, gui: ScreenGui)
    menus[name] = gui
    gui.Enabled = false
end

function MenuManager.Open(name: string)
    local gui = menus[name]; if not gui then return end

    -- Close current major menu (if any)
    if current and current ~= name then
        menus[current].Enabled = false
        fireListeners(current, false)
    end

    current = name
    gui.Enabled = true
    fireListeners(name, true)
end

function MenuManager.Close(name: string?)
    name = name or current
    if not name then return end
    if menus[name] then menus[name].Enabled = false end
    fireListeners(name, false)
    if current == name then current = nil end
end

function MenuManager.Toggle(name: string)
    if current == name then MenuManager.Close(name)
    else MenuManager.Open(name) end
end

-- Modal stack (dialogs over current menu)
function MenuManager.PushModal(name: string)
    local gui = menus[name]; if not gui then return end
    table.insert(stack, name)
    gui.Enabled = true
    fireListeners(name, true)
end

function MenuManager.PopModal(): string?
    local name = table.remove(stack)
    if name and menus[name] then
        menus[name].Enabled = false
        fireListeners(name, false)
    end
    return name
end

function MenuManager.IsOpen(name: string): boolean
    return current == name or table.find(stack, name) ~= nil
end

function MenuManager.OnChanged(name: string, fn: (open: boolean) -> ())
    listeners[name] = listeners[name] or {}
    table.insert(listeners[name], fn)
end

local function fireListeners(name: string, open: boolean)
    for _, fn in (listeners[name] or {}) do pcall(fn, open) end
end

return MenuManager
```

This is the contract every menu observes. No menu manages its own visibility directly; it asks `MenuManager` to open/close it, which guarantees mutual exclusion at the major level and stacking at the modal level.

## Hotkey-driven toggling

Bind menu hotkeys via `ContextActionService` (covered in [input-handling.md](input-handling.md)) — central, gamepad-friendly, supports rebinding:

```lua
local CAS = game:GetService("ContextActionService")
local MenuManager = require(game.ReplicatedStorage.UI.MenuManager)

local function bindToggle(action: string, menu: string, ...)
    CAS:BindAction(action, function(_, state)
        if state ~= Enum.UserInputState.Begin then return end
        MenuManager.Toggle(menu)
        return Enum.ContextActionResult.Sink
    end, false, ...)
end

bindToggle("ToggleInventory", "InventoryMenu", Enum.KeyCode.I, Enum.KeyCode.ButtonY)
bindToggle("ToggleMap",       "MapMenu",       Enum.KeyCode.M)
bindToggle("TogglePause",     "PauseMenu",     Enum.KeyCode.Escape, Enum.KeyCode.ButtonStart)
```

For ESC behavior with a back-stack (ESC closes current modal first, then current menu, then opens pause if nothing was open):

```lua
local function handleEscape()
    if #stack > 0 then MenuManager.PopModal()
    elseif current then MenuManager.Close(current)
    else MenuManager.Open("PauseMenu") end
end

CAS:BindAction("Escape", function(_, state)
    if state == Enum.UserInputState.Begin then handleEscape() end
    return Enum.ContextActionResult.Sink
end, false, Enum.KeyCode.Escape)
```

This is the "expected" ESC behavior in nearly every modern game.

## Open/close transitions

A menu that snaps in/out feels cheap. Pair `MenuManager.OnChanged` with TweenService for slide-in / fade-in:

```lua
MenuManager.OnChanged("PauseMenu", function(open)
    local frame = pauseGui.Container
    local startPos = open and UDim2.fromScale(0.5, -0.5) or UDim2.fromScale(0.5, 0.5)
    local endPos = open and UDim2.fromScale(0.5, 0.5) or UDim2.fromScale(0.5, -0.5)
    frame.Position = startPos
    TweenService:Create(frame,
        TweenInfo.new(0.25, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
        {Position = endPos}
    ):Play()
end)
```

For close transitions, run the close tween *before* setting `Enabled = false`. To do this, MenuManager.OnChanged could hand off the actual disable to the consumer — or the consumer can manage its own visibility and just respect the canonical open/close events.

See [tweens-animation.md](tweens-animation.md) for easing presets.

## Tab system pattern

A settings menu has Audio / Graphics / Controls tabs. The classic pattern: tab buttons at top, content panel below, only one tab's content visible at a time.

```lua
-- Each tab has a ButtonFrame (in a row) and a ContentFrame (in a stack)
local tabs = {
    {id = "Audio", button = audioBtn, content = audioContent},
    {id = "Graphics", button = graphicsBtn, content = graphicsContent},
    {id = "Controls", button = controlsBtn, content = controlsContent},
}

local activeTab: string? = nil

local function setActiveTab(id: string)
    activeTab = id
    for _, tab in tabs do
        tab.content.Visible = (tab.id == id)
        tab.button.BackgroundColor3 = (tab.id == id)
            and Color3.fromRGB(80, 120, 200)
            or Color3.fromRGB(50, 50, 50)
    end
end

for _, tab in tabs do
    tab.button.Activated:Connect(function() setActiveTab(tab.id) end)
end

setActiveTab("Audio")    -- default
```

For deeper hierarchies (Settings → Audio → Master Volume slider), use the same MenuManager pattern recursively, or treat sub-tabs as plain visibility toggles within the current menu.

## Settings persistence

Settings (volume, FOV, key bindings, graphics quality) typically live in DataStore so they follow the player across sessions. Pattern:

```lua
-- ServerStorage/Settings.lua
local Settings = {}

local DEFAULT = {
    masterVolume = 0.8,
    musicVolume = 0.5,
    sfxVolume = 0.8,
    fov = 70,
    cameraSensitivity = 1,
    graphicsQuality = "High",
}

function Settings.Load(player: Player): typeof(DEFAULT)
    -- Load from DataStore profile (see datastores.md)
    local profile = PlayerData.Get(player)
    return profile and profile.settings or table.clone(DEFAULT)
end

function Settings.Update(player: Player, patch: {[string]: any})
    local profile = PlayerData.Get(player); if not profile then return end
    profile.settings = profile.settings or table.clone(DEFAULT)
    for k, v in patch do
        if DEFAULT[k] ~= nil then    -- only allow known keys
            profile.settings[k] = v
        end
    end
    SettingsUpdatedRemote:FireClient(player, profile.settings)
end

return Settings
```

The client-side settings menu reads these on open; on each slider/toggle change, fire `SettingsUpdateRemote` to the server with the patch. Apply settings client-side immediately for responsive feel.

For settings that affect game systems (volume → SoundService.Volume; sensitivity → camera mouse-delta multiplier), wire the `OnChanged` of each setting to its consumer.

## Confirmation dialogs

A modal "Are you sure?" pop-up that blocks input until answered. Push onto the modal stack:

```lua
local function showConfirm(title: string, body: string, onYes: () -> (), onNo: (() -> ())?)
    confirmGui.Title.Text = title
    confirmGui.Body.Text = body

    local yesConn, noConn
    yesConn = confirmGui.YesButton.Activated:Connect(function()
        yesConn:Disconnect(); noConn:Disconnect()
        MenuManager.PopModal()
        onYes()
    end)
    noConn = confirmGui.NoButton.Activated:Connect(function()
        yesConn:Disconnect(); noConn:Disconnect()
        MenuManager.PopModal()
        if onNo then onNo() end
    end)

    MenuManager.PushModal("ConfirmDialog")
end

-- Usage
showConfirm("Delete save slot?",
    "This will permanently remove this character. This cannot be undone.",
    function() deleteSlot(slotIndex) end)
```

For richer dialogs (Yes / No / Cancel; OK only; text-input prompt), parameterize `showConfirm` with button definitions.

## Loading screen

For places with significant load time (cross-place teleport, large places), show a loading screen *before* the default UI loads. Parent a script + UI under **`ReplicatedFirst`** — these run before everything else and replace Roblox's default loading bar.

```lua
-- ReplicatedFirst/LoadingScreen (LocalScript)
local Players = game:GetService("Players")
local ReplicatedFirst = game:GetService("ReplicatedFirst")

ReplicatedFirst:RemoveDefaultLoadingScreen()

local screen = Instance.new("ScreenGui")
screen.IgnoreGuiInset = true
screen.Parent = Players.LocalPlayer:WaitForChild("PlayerGui")

local frame = Instance.new("Frame", screen)
frame.Size = UDim2.fromScale(1, 1)
frame.BackgroundColor3 = Color3.new(0, 0, 0)

local label = Instance.new("TextLabel", frame)
label.Text = "Loading..."
-- ... etc.

if not game:IsLoaded() then game.Loaded:Wait() end
task.wait(0.5)    -- brief grace
screen:Destroy()
```

For more polish: progress bar driven by `ContentProvider:PreloadAsync` to preload assets explicitly.

See [project-structure.md](project-structure.md) for `ReplicatedFirst`.

## Tooltips

Hover a UI element → small panel appears near the cursor with details. Pattern:

```lua
local tooltip = playerGui:WaitForChild("Tooltip")
tooltip.Enabled = false

local function attachTooltip(gui: GuiObject, getText: () -> string)
    gui.MouseEnter:Connect(function()
        tooltip.Label.Text = getText()
        tooltip.Enabled = true
    end)
    gui.MouseMoved:Connect(function(x, y)
        tooltip.Container.Position = UDim2.fromOffset(x + 16, y + 16)
    end)
    gui.MouseLeave:Connect(function()
        tooltip.Enabled = false
    end)
end
```

For touch-driven tooltips (no MouseEnter), use long-press or an info icon (`ⓘ`) the user taps to toggle.

For inventory-item tooltips with rich content (icon, stats, rarity), build the tooltip Frame as a template; `getText` becomes `getItemTooltipData` returning the data the tooltip renders.

## HUD vs Menu

| HUD | Menu |
|---|---|
| Always visible during gameplay | Toggleable; usually pauses/dims gameplay |
| Small, peripheral (corners, edges) | Center-stage; significant screen real estate |
| Read-only display (health, ammo, minimap) | Interactive (buttons, sliders, scrolling lists) |
| `ScreenGui` parented to PlayerGui at start | `ScreenGui` registered with MenuManager |

HUD elements sit in their own `ScreenGui` (separate from the Menus container). Examples already covered:
- Leaderstats top-right scoreboard — see [leaderstats.md](leaderstats.md)
- Ammo counter — see [guns-fps.md](guns-fps.md)
- Status-effect icons — see [status-effects.md](status-effects.md)
- Toast notifications — see [live-ops.md](live-ops.md)

## Mobile considerations

- **Touch-first menus need bigger hit targets** (44pt minimum per Apple HIG; Roblox-equivalent ~48px).
- **Hotkeys don't work on mobile.** Always provide an on-screen menu button.
- **Long lists need explicit scroll affordance.** A `ScrollingFrame.ScrollBarThickness` of 6 is invisible on touch — use 12+ or rely on intuitive vertical drag.
- **`MouseEnter`/`MouseLeave`** don't fire on touch. Tooltips need an alternate trigger (info icon, long-press).
- **Auto-scale UI on small screens.** Use `UIScale` keyed off `workspace.CurrentCamera.ViewportSize` to shrink menus on smaller devices.

```lua
local function scaleForViewport(uiScale: UIScale)
    local h = workspace.CurrentCamera.ViewportSize.Y
    uiScale.Scale = math.clamp(h / 900, 0.6, 1.2)
end
```

See [input-handling.md](input-handling.md) for touch / cross-platform input patterns.

## Pausing the game while a menu is open

In a single-player or co-op game with no real "server pause," opening the pause menu can:
- Set `Workspace.Gravity = 0` (freeze physics) — affects everyone
- Set the player's `humanoid.WalkSpeed = 0` (prevent the local player moving) — only affects them
- Display a UI overlay that absorbs input

For multiplayer, you usually **can't really pause** (other players are still playing). The "pause menu" is just an overlay that's open for the local player; gameplay continues server-side. Be honest with the player ("Game continues while this menu is open").

## Patterns

### Main menu / title screen
Show before the player is in-game. Often the menu opens a `MainMenu` ScreenGui covering the full screen, with "Play", "Settings", "Quit" buttons. "Play" closes the menu and reveals the world (and possibly fires a server RemoteEvent so the server knows to spawn the character). For multi-slot games, "Play" opens the slot picker (see [save-slots.md](save-slots.md)).

```lua
playButton.Activated:Connect(function()
    MenuManager.Close("MainMenu")
    PlayRequestRemote:FireServer()        -- server spawns character now
end)
```

If you want to delay character spawning until the player picks "Play," set `Players.CharacterAutoLoads = false` on the server, then `:LoadCharacter()` on the request. See [player-lifecycle.md](player-lifecycle.md).

### Quit-to-menu
"Return to main menu" from in-game. Closes all menus, hides HUD, opens MainMenu, optionally despawns character or resets state.

### Character / class select
A focused menu that lets the player pick from a list (e.g., classes in a class-based shooter). Each entry shows preview (model in a `ViewportFrame`), description, stats. On confirm, fire RemoteEvent with the choice.

### Inventory + shop in the same place
For RPGs, the inventory menu often has a "Shop" sub-tab when near a vendor. Implement as `InventoryMenu` with tabs; the Shop tab is hidden unless a vendor is in range. Same MenuManager.Open call; tab logic gates the visibility.

### Wheel / radial menus (controller-friendly)
Pet-equip / emote-pick wheel: 8 segments arranged radially. On gamepad right-stick, the angle determines selection; on release, fire the chosen action. Custom render with TweenService for smooth highlights.

### Drag and drop
For inventory drag-to-equip:
- Use `UIDragDetector` (modern Roblox primitive) or roll your own with `UserInputService.InputChanged` and a "ghost" frame following the cursor.
- On drop, hit-test against valid drop targets (use `gui:GetGuiObjectsAtPosition()` for accurate detection).

## Pitfalls

- **No central state manager.** Each menu sets `Enabled = true` directly. Two menus end up open; ESC doesn't know which to close. Always go through MenuManager.
- **`ResetOnSpawn = true` (default) on persistent UIs.** Inventory/HUD wiped on every respawn. Set `false`.
- **Menu open while gameplay continues without indication.** Players in multiplayer think the game paused; they didn't. Show a "Game continues" badge.
- **Hotkey toggles a menu while typing in a TextBox.** I-key opens inventory while you're typing "Hi" → unexpected. Check `gameProcessed` / `:IsFocused()` on TextBoxes.
- **Tab system without persisting active tab.** Player navigates back to settings, has to re-pick Audio every time. Persist last-viewed tab.
- **Settings applied client-side but not saved.** Player adjusts volume, refreshes the game, volume is back to default. Save on every change OR on menu close.
- **Settings save fired per-keystroke on a slider.** 60 saves per second when dragging. Debounce — save on Released or after 0.3s of inactivity.
- **Confirmation dialog button connections leak.** Each `showConfirm` adds a new connection to the same buttons; old confirmations still fire. Disconnect on dismiss.
- **Modal stack never popped on menu close.** Player closes the parent menu but a tooltip or confirm dialog stays visible forever. Pop on parent close.
- **Loading screen stays after the place loads** because you forgot to `:Destroy()` it. Always destroy after `game.Loaded:Wait()`.
- **Tooltip flickers near screen edge** because Position pushes it offscreen and re-triggers MouseLeave. Clamp tooltip position so it stays on-screen.
- **Mobile UI usable on PC but not vice versa.** Test both. ScreenGui scaling and AnchorPoint discipline matter most for cross-device.
- **Server-driven menu open.** `OpenMenuRemote:FireClient(player, "Inventory")` from the server is fine for "shop NPC opens shop UI"; from the server forcing menus open at random is jarring. Save server-driven opens for genuine triggers.
- **HUD UIs in the Menus container.** ESC accidentally closes them or they get hidden when a menu opens. Keep HUD in its own ScreenGui, separate from menus.

## See also
- [ui-system.md](ui-system.md) — the primitives this card composes
- [input-handling.md](input-handling.md) — ContextActionService for hotkey toggles, cross-platform menu input
- [tweens-animation.md](tweens-animation.md) — open/close transitions
- [datastores.md](datastores.md) — settings persistence
- [player-lifecycle.md](player-lifecycle.md) — `CharacterAutoLoads = false` for main-menu-first games; PlayerGui setup on join
- [project-structure.md](project-structure.md) — StarterGui auto-copy, ReplicatedFirst loading screens
- [leaderstats.md](leaderstats.md), [guns-fps.md](guns-fps.md), [status-effects.md](status-effects.md), [live-ops.md](live-ops.md) — HUD elements that live alongside (not inside) the menu system
- [inventory.md](inventory.md), [shops.md](shops.md), [quests.md](quests.md), [skill-trees.md](skill-trees.md), [save-slots.md](save-slots.md) — common menus this manager opens
- [save-slots.md](save-slots.md) — slot picker is a major menu before character spawn
- [community-libraries.md](community-libraries.md) — **Roact / React-Lua** for declarative UI; **TopbarPlus** for menu icons in the top bar
