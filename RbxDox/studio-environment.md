# Studio Environment

> Upstream: <https://create.roblox.com/docs/studio>
> Source: `content/en-us/studio/explorer.md`, `properties.md`, `output.md`, `script-editor.md`, `testing-modes.md`

## Purpose
Roblox Studio is the IDE for building experiences. The five panels you'll touch every minute: Explorer, Properties, Output, Script Editor, and the Test toolbar.

## Explorer
Tree view of the DataModel. `View → Explorer` (Ctrl/Cmd+1).
- Click an Instance to select; `+` icon adds a child by class.
- Right-click → **Save to File / Insert from File** for `.rbxm` round-tripping.
- **Filter** box at the top supports class names (`Part`), exact names, and tags (`#mytag`).
- Hold `Alt` and click a result in Output to jump to the offending Instance.

## Properties
Shows everything editable on the selected Instance. `View → Properties` (Ctrl/Cmd+2).
- Grouped by category (Appearance, Behavior, Data, etc.).
- Bottom **Attributes** section is for custom designer-tunable values — see [attributes.md](attributes.md).
- Multi-select multiple Instances to bulk-edit common properties.
- Property names here match the runtime API exactly: `BrickColor`, `Anchored`, `CFrame`, etc.

## Output
Console for `print`, warnings, and errors. `View → Output` (Ctrl/Cmd+8).
- Errors are clickable → jumps to the script and line.
- Filter buttons toggle Print / Info / Warning / Error.
- `Ctrl/Cmd+L` clears.
- The **Command Bar** (`View → Command Bar`) below executes one-shot Luau against the running session — useful for inspecting state without editing a script.

## Script Editor
Tabbed Luau editor.
- **Autocomplete** is type-driven. Annotate parameters/returns to get richer suggestions (see [luau-types.md](luau-types.md)).
- `F2` to rename a symbol; `F12` to go to definition.
- The **Script Analysis** pane (`View → Script Analysis`) lists static issues without running.
- **Breakpoints** by clicking the gutter; the debugger pauses on hit.
- **Find/Replace** is `Ctrl/Cmd+F` / `+H`; `Ctrl/Cmd+Shift+F` searches across all scripts in the place.

## Testing modes
The **Test** tab (or `F5`).

| Mode | What it simulates |
|---|---|
| **Play** (`F5`) | Spawns local server + one client (your character) |
| **Play Here** | Same but spawns at the camera, not the SpawnLocation |
| **Run** (`F8`) | Server only — no character, no LocalScripts |
| **Start Server / Players** | Real local server + N separate Studio windows as clients (test multiplayer) |

`F5` covers most workflows. Use **Start Server with N players** when debugging anything client/server interaction-heavy.

## Other useful panels
- **View → Terrain Editor** — paint, sculpt, region tools.
- **View → Toolbox** — Marketplace assets, your inventory, packages.
- **View → Asset Manager** — places, images, meshes, audio uploaded to your experience.
- **View → Game Explorer** — places and shared modules of the experience.
- **View → MicroProfiler** (`Ctrl/Cmd+F6`) — frame-by-frame perf — see [performance.md](performance.md).
- **View → Developer Console** (`F9` in Play mode) — runtime client console; players can open this too.

## Pitfalls

- **Unsaved play-test changes.** Edits made *during* Play mode are reverted when you stop. Toggle the "Edit Mode" indicator carefully — it's red while playing.
- **Default cmd-bar runs in client context.** When in Play mode, switch the dropdown to "Server" to inspect server state.
- **Two scripts with the same name.** Allowed and confusing. Rename to be unique within a parent.
- **Disabled vs deleted scripts.** `Disabled = true` keeps the script in the tree but doesn't run; useful for toggling without losing the file.

## See also
- [studio-mcp.md](studio-mcp.md) — controlling Studio from Claude via MCP
- [scripts-and-modules.md](scripts-and-modules.md) — script types
- [performance.md](performance.md) — MicroProfiler
