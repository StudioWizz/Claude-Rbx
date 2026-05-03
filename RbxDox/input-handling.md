# Input Handling

> Upstream: <https://create.roblox.com/docs/input>
> Source: `content/en-us/input/index.md`, `mouse-and-keyboard.md`, `gamepad.md`, `mobile.md`, `input-action-system.md`

## Purpose
All input is **client-side**. Two services to know:
- **`UserInputService`** — low-level: every key, every mouse move, every touch, every gamepad button. Use for cameras, custom UI, raw event handling.
- **`ContextActionService`** — bind named actions to inputs and (optionally) auto-create touch buttons on mobile and gamepad bindings. Use for game actions ("Jump", "Reload", "Sprint").

Both are LocalScript-only. The server never sees raw input.

## UserInputService

```lua
local UIS = game:GetService("UserInputService")

UIS.InputBegan:Connect(function(input: InputObject, gameProcessed: boolean)
    if gameProcessed then return end   -- chat/UI consumed it
    if input.KeyCode == Enum.KeyCode.E then doInteract() end
    if input.UserInputType == Enum.UserInputType.MouseButton1 then onClick() end
end)

UIS.InputChanged:Connect(function(input, gameProcessed) end)  -- mouse moves, gamepad sticks
UIS.InputEnded:Connect(function(input, gameProcessed) end)

UIS:IsKeyDown(Enum.KeyCode.W)
UIS:IsMouseButtonPressed(Enum.UserInputType.MouseButton1)
UIS:GetMouseLocation()                  -- screen space
UIS:GetMouseDelta()                     -- per-frame delta (when MouseBehavior locks)
UIS.MouseBehavior = Enum.MouseBehavior.LockCenter   -- FPS-style lock
UIS.MouseIconEnabled = true
```

Detect platform:
```lua
UIS.TouchEnabled       -- mobile/tablet
UIS.KeyboardEnabled
UIS.MouseEnabled
UIS.GamepadEnabled
```

## ContextActionService

Bind a single named action across all input devices:

```lua
local CAS = game:GetService("ContextActionService")

local function onJump(actionName, inputState, inputObject)
    if inputState == Enum.UserInputState.Begin then
        humanoid.Jump = true
    end
    return Enum.ContextActionResult.Sink   -- or Pass to bubble
end

CAS:BindAction("Jump", onJump, true,           -- 3rd arg: createTouchButton
    Enum.KeyCode.Space,                         -- keyboard
    Enum.KeyCode.ButtonA,                       -- gamepad
    Enum.UserInputType.MouseButton2)            -- mouse

CAS:SetTitle("Jump", "Jump")
CAS:SetImage("Jump", "rbxassetid://...")        -- mobile button icon
CAS:SetPosition("Jump", UDim2.fromScale(0.8, 0.8))

-- Later
CAS:UnbindAction("Jump")
```

`inputState` cycles `Begin → Change → End`. Return `Sink` to consume, `Pass` to let other handlers see it.

When `createTouchButton = true` and the user is on a touch device, ContextActionService auto-creates an on-screen button — instant mobile support.

## Gamepad

Buttons: `ButtonA/B/X/Y`, `ButtonL1/L2/R1/R2`, `ButtonStart/ButtonSelect`, `DPadUp/Down/Left/Right`, `ButtonL3/R3` (stick clicks), `Thumbstick1/2`.

Sticks come through `InputChanged` with `KeyCode = Thumbstick1/2` and `Position` as a Vector3 in [-1, 1]:
```lua
UIS.InputChanged:Connect(function(input)
    if input.KeyCode == Enum.KeyCode.Thumbstick1 then
        local dir = Vector2.new(input.Position.X, input.Position.Z)
        if dir.Magnitude > 0.1 then move(dir) end   -- deadzone
    end
end)
```

Gamepad detection: `UIS:GetConnectedGamepads()`, `UIS.GamepadConnected`/`Disconnected`.

## Touch

```lua
UIS.TouchStarted:Connect(function(touch, gameProcessed) end)
UIS.TouchMoved:Connect(function(touch, gameProcessed) end)
UIS.TouchEnded:Connect(function(touch, gameProcessed) end)

UIS.TouchTap:Connect(function(positions, gameProcessed) end)
UIS.TouchPinch:Connect(function(positions, scale, velocity, state, gameProcessed) end)
UIS.TouchRotate:Connect(function(positions, rotation, velocity, state, gameProcessed) end)
UIS.TouchSwipe:Connect(function(direction, count, gameProcessed) end)
```

`TouchEnabled` should drive UI scaling and on-screen control rendering decisions.

## Patterns

### Sprint while holding shift
```lua
CAS:BindAction("Sprint", function(_, state)
    if state == Enum.UserInputState.Begin then
        humanoid.WalkSpeed = 24
    elseif state == Enum.UserInputState.End then
        humanoid.WalkSpeed = 16
    end
end, false, Enum.KeyCode.LeftShift, Enum.KeyCode.ButtonL3)
```

### Click-to-shoot (with server validation)
```lua
local Mouse = game.Players.LocalPlayer:GetMouse()  -- legacy but still useful
UIS.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        local target = workspace:Raycast(camera.CFrame.Position, Mouse.UnitRay.Direction * 1000)
        ShootRemote:FireServer(target and target.Position)
    end
end)
```
Server re-raycasts to validate; never trust the client's claim of what it hit.

### Cross-platform jump
```lua
CAS:BindAction("Jump", function(_, s)
    if s == Enum.UserInputState.Begin then humanoid.Jump = true end
end, true, Enum.KeyCode.Space, Enum.KeyCode.ButtonA)
```

## Pitfalls

- **Ignoring `gameProcessed`.** When the chat or a UI element consumes input, you'll fire your action AND chat — annoying. Always early-return.
- **Polling input every frame** when an event would do. `IsKeyDown` is fine occasionally, but per-frame loops eat CPU.
- **Not handling gamepad/touch.** Roblox's audience is heavily mobile and console. Even a simple click should also have a touch path.
- **Locking the mouse without releasing.** `LockCenter` traps the cursor; users can't click your UI. Toggle off when opening menus.
- **Rebinding actions you didn't unbind.** Pile-ups. Use one CAS BindAction per action name, or Unbind first.
- **Touching input on the server.** Won't work. Send a remote.

## See also
- [security.md](security.md) — never trust client input
- [camera.md](camera.md) — mouse-driven cameras
- [ui-system.md](ui-system.md) — buttons consume input automatically
- [menu-systems.md](menu-systems.md) — ContextActionService for menu hotkey toggles (ESC, I, M)
