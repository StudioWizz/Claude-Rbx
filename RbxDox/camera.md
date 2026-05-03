# Camera

> Upstream: <https://create.roblox.com/docs/workspace/camera>
> Source: `content/en-us/workspace/camera.md`

## Purpose
Each client has exactly one active `Camera`, accessible as `workspace.CurrentCamera`. The camera's `CFrame` is what the player sees. By default Roblox provides a mouse-orbit/follow camera tracking the character's HumanoidRootPart, but you can override this fully.

## Camera basics

```lua
local cam = workspace.CurrentCamera

cam.CFrame                  -- where the camera is and where it's looking
cam.Focus                   -- the point the default scripts orbit
cam.FieldOfView             -- vertical FOV degrees, default 70
cam.CameraType              -- Custom | Scriptable | Track | Attach | Watch | Fixed | Follow
cam.CameraSubject           -- usually the player's Humanoid

cam:WorldToScreenPoint(worldPos)        -- → (Vector3 (x, y, depth), boolean visible)
cam:WorldToViewportPoint(worldPos)      -- same but viewport-space
cam:ViewportPointToRay(x, y, depth?)    -- → Ray for mouse picking
cam:ScreenPointToRay(x, y)
```

## CameraType values

| CameraType | Behaviour |
|---|---|
| `Custom` (default) | Engine-driven, follows CameraSubject with mouse orbit |
| `Scriptable` | Engine does nothing; you set CFrame every frame |
| `Track` | Follows subject, no orbit |
| `Attach` | Locked to subject's CFrame |
| `Watch` | Follows subject as a target, camera stays in place |
| `Fixed` | Doesn't move |
| `Follow` | Follows subject's position from above |

For a fully custom camera, set `CameraType = Scriptable` and drive `CFrame` from `RenderStepped`.

## Patterns

### Custom orbit camera
```lua
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local cam = workspace.CurrentCamera
cam.CameraType = Enum.CameraType.Scriptable

local yaw, pitch = 0, math.rad(-15)
local distance = 15

UIS.InputChanged:Connect(function(input, gp)
    if gp then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement and UIS:IsMouseButtonPressed(Enum.UserInputType.MouseButton2) then
        yaw -= math.rad(input.Delta.X * 0.4)
        pitch = math.clamp(pitch - math.rad(input.Delta.Y * 0.4), math.rad(-80), math.rad(80))
    end
end)

RunService.RenderStepped:Connect(function()
    local char = game.Players.LocalPlayer.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local pivot = CFrame.new(hrp.Position) * CFrame.fromEulerAnglesYXZ(pitch, yaw, 0)
    cam.CFrame = pivot * CFrame.new(0, 0, distance)
end)
```

### First-person lock
```lua
local Players = game:GetService("Players")
Players.LocalPlayer.CameraMode = Enum.CameraMode.LockFirstPerson
```

### Smooth follow with lookahead
```lua
RunService.RenderStepped:Connect(function(dt)
    local target = subject.Position + subject.AssemblyLinearVelocity * 0.2
    local desired = CFrame.lookAt(target + Vector3.new(0, 5, 15), target)
    cam.CFrame = cam.CFrame:Lerp(desired, math.clamp(dt * 5, 0, 1))
end)
```

### Mouse-pick a part
```lua
local UIS = game:GetService("UserInputService")

local function pick()
    local mouse = UIS:GetMouseLocation()
    local ray = cam:ViewportPointToRay(mouse.X, mouse.Y)
    return workspace:Raycast(ray.Origin, ray.Direction * 1000)
end
```

### World-space UI label that tracks an object
```lua
RunService.RenderStepped:Connect(function()
    local screenPos, onScreen = cam:WorldToViewportPoint(target.Position)
    label.Position = UDim2.fromOffset(screenPos.X, screenPos.Y)
    label.Visible = onScreen
end)
```

## Cinematic camera (cutscenes)

```lua
local TweenService = game:GetService("TweenService")
cam.CameraType = Enum.CameraType.Scriptable

local info = TweenInfo.new(2, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut)
TweenService:Create(cam, info, {CFrame = endCFrame}):Play()
```

When the cutscene ends, restore:
```lua
cam.CameraType = Enum.CameraType.Custom
cam.CameraSubject = humanoid
```

## Pitfalls

- **Setting `cam.CFrame` from a `Script` (server).** `CurrentCamera` is per-client; only LocalScripts should drive it.
- **Forgetting to set `CameraType = Scriptable`** when overriding CFrame — the default scripts will fight you.
- **Using `Stepped`/`Heartbeat` for camera** — your camera will lag a frame behind the render. Always use `RenderStepped`.
- **Not restoring CameraType** after a cutscene. Players get stuck in fixed view.
- **`FieldOfView` extremes.** Below 10 looks like a sniper scope; above 100 distorts heavily. 70–90 is the comfort range.
- **Camera CFrame inside a part.** Triggers near-clipping. Check the camera isn't stuck inside geometry.

## See also
- [cframes-vectors.md](cframes-vectors.md) — CFrame math
- [input-handling.md](input-handling.md) — mouse delta with locked cursor
- [tweens-animation.md](tweens-animation.md) — TweenService
