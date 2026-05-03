# Roblox Studio MCP

> Source: the Roblox Studio MCP proxy bridges Claude with a running Studio instance. Tools are surfaced under the `mcp__Roblox_Studio__*` namespace.

## Purpose
This environment connects Claude to a live Roblox Studio session over MCP. That means Claude can read the current DataModel, run Luau against it, edit scripts, capture screenshots, even drive play-test input — all from this conversation. Use these tools to inspect and iterate on the live place rather than only writing files to disk.

## Workflow at the start of every session

1. `list_roblox_studios` — see what Studio instances are connected. Each has an `id`, `name`, and `active` flag.
2. `set_active_studio` — pick one as the target for subsequent calls. Only needed when there are multiple Studio windows or none is yet active.
3. From here, every other Studio tool acts on the active studio.

## Tool reference

### Inspection
- **`list_roblox_studios`** — list all connected Studio sessions.
- **`set_active_studio(studio_id)`** — make `studio_id` the target.
- **`inspect_instance(path)`** — dump properties, attributes, children of an Instance by DataModel path (e.g. `game.Workspace.Baseplate`).
- **`search_game_tree(query)`** — search the DataModel for Instances by name/class.
- **`screen_capture()`** — screenshot the Studio viewport. Useful for visual confirmation after edits.

### Code
- **`execute_luau(code)`** — run arbitrary Luau in Studio's command-bar context. Returns the value of the final expression (use `return ...` to make it explicit). Yields the way the command bar does. Great for one-shot mutations and read-only queries.
- **`script_read(path)`** — read the source of a Script/LocalScript/ModuleScript at the given DataModel path.
- **`script_grep(pattern)`** — regex over all script source in the place.
- **`script_search(query)`** — semantic-style search across scripts.
- **`multi_edit(...)`** — apply many edits to scripts in one transaction.

### Assets
- **`search_creator_store(query)`** / **`insert_from_creator_store(asset_id)`** — find and insert Marketplace assets.
- **`store_image(...)`**, **`generate_material(...)`**, **`generate_mesh(...)`** — author new assets.
- **`generate_material`/`mesh`** uses Roblox's generative pipeline — verify visually before committing.

### Play-test control
- **`start_stop_play()`** — start/stop play-test (the F5 button).
- **`get_console_output()`** — the runtime Output panel's contents. Read after running code or play-testing.
- **`character_navigation(...)`** — drive the character avatar during play-test.
- **`user_keyboard_input(...)` / `user_mouse_input(...)`** — synthetic input, useful for testing UI flows.

### Subagents
- **`subagent(...)`** — spawn an in-Studio subagent for parallel/long-running work inside Studio. Niche; the standard `Agent` tool is usually preferable.

## Patterns

### Spawn a part
```
execute_luau:
  local p = Instance.new("Part")
  p.Size = Vector3.new(4,1,4); p.Anchored = true
  p.Position = Vector3.new(0, 5, 0); p.Parent = workspace
  return p:GetFullName()
```

### Read what's in Workspace
```
search_game_tree("Workspace")          -- top-level overview
inspect_instance("game.Workspace.Baseplate")
```

### Find usages of a function
```
script_grep("DataStoreService:GetDataStore")
```

### Edit a script then verify
```
script_read("game.ServerScriptService.Main")        -- get current source
multi_edit(...)                                      -- apply changes
get_console_output()                                 -- check there are no parse errors
```

### Visual check after a UI change
```
start_stop_play()     -- enter play mode
screen_capture()      -- look at the result
start_stop_play()     -- back to edit mode
```

## Best practices for Claude using this MCP

- **Always check `list_roblox_studios` first** in a new session to confirm which Studio is connected. Don't assume the previous session's active studio is still valid.
- **`execute_luau` for read-only queries first.** Inspect before mutating; it's free to undo a missed assumption when you haven't changed anything yet.
- **Wrap mutations in `pcall`** inside `execute_luau` — errors are returned but printed errors are easier to diagnose than silent failures.
- **Print or `return` data** explicitly. The command bar context returns the last expression; without `return`, you get nothing useful back.
- **For non-trivial edits, prefer `script_read` → `multi_edit`** over `execute_luau` mutations of source — it produces a real diff in the place.
- **Don't blast play-test input** at a stopped session. Check play state first.
- **Studio is a shared workspace.** The user is watching. Keep mutations small and reversible; ask before destructive changes (deleting many Instances, overwriting scripts).
- **`screen_capture` is cheap** — use it to confirm visual outcomes, especially after CFrame/UI changes.

## Pitfalls

- **`execute_luau` runs in the Studio process**, not at the server or client of a play-test. To run code in a play session, prefer scripts you've placed and rely on play-test, or use the cmd bar dropdown to switch context.
- **Edits during play-test are reverted on stop.** Make persistent changes from edit mode.
- **`get_console_output` is the runtime console**, not your `execute_luau` return value. Each tool returns its own results.
- **Many tools yield.** Don't expect millisecond round-trips; design prompts to handle the latency.
- **`generate_material` / `generate_mesh`** can produce off-style assets. Always preview before committing to a build.

## See also
- [studio-environment.md](studio-environment.md) — the same Studio panels by hand
- [procedural-construction.md](procedural-construction.md) — patterns for using `execute_luau` + `screen_capture` to build structures
- All other RbxDox cards — tell Claude which APIs to invoke via `execute_luau`
