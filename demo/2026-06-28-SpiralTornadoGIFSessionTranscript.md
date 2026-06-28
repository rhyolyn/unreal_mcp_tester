# Session Transcript — Spiral Tornado GIF

**Date:** 2026-06-28
**Project:** Unreal MCP Tester (`C:\Users\rhyol\Documents\Unreal\mcp_tester`)
**Engine:** Unreal Engine 5.8 (experimental ModelContextProtocol plugin)
**Tooling:** Claude Code (Opus 4.8) driving the editor over MCP

---

## Overview

In this session we:

1. Recorded a full revolution of the sphere-tornado animation and produced a
   looping animated GIF.
2. Moved the GIF into a `demo/` directory and referenced it from `README.md`.
3. Worked out how to persist the editor's MCP + Terminal settings so they
   stop resetting between sessions.
4. Wrote this transcript.

---

## 1. "Play the level, start the animation, record the spiral, make a GIF"

### Getting connected

- At first the Unreal MCP tools were **not** loaded in the session. The editor
  process (`UnrealEditor.exe`) was running, but the MCP HTTP server that
  `.mcp.json` points at (`http://127.0.0.1:8000/mcp`) was not responding — the
  editor was only listening on port **1985**.
- Reported the blocker. The user **started the MCP server**, after which the
  `unreal-mcp` toolsets registered (`list_toolsets`, `describe_toolset`,
  `call_tool`).

### Understanding the animation

- Inspected `BP_SphereTornado`:
  - Variables: `Spinning` (bool), `Angle`, `SpinSpeed`, `Radius`.
  - `EventBeginPlay` sets `SpinSpeed = 30`, `Radius = 200`.
  - `EventTick`: pressing **G** toggles `Spinning`; while spinning,
    `Angle += SpinSpeed * DeltaSeconds` and every actor tagged `TornadoSphere`
    is repositioned via `MoveSphere`.
  - **`MoveSphere` is a pure, absolute function of angle + height:**
    `pos = (Radius·cos(Angle + h·360/7), Radius·sin(Angle + h·360/7), z)`,
    where `h = (z − 50)/100` is the sphere's height index (0–6).
  - Therefore **one full revolution = 360 / 30 = 12 seconds**, and each of the
    7 spheres is offset by `360/7 ≈ 51.43°` (a helix).

### Capturing — three obstacles and the fixes

1. **Framing.** The level-viewport camera pointed at empty desert. Found the
   tower at world origin `(0,0)`, cubes stacked `z = 0…700`, spheres orbiting
   at radius 200. Chose a fixed 3/4 capture pose:
   `location (-750,-750,520)`, `rotation (pitch -8, yaw 45)`.

2. **Starting the spin.** `PressKey "G"` did nothing until the PIE viewport was
   given focus (clicked the viewport widget `i3`, then `G`). Setting
   `Spinning`/`SpinSpeed`/`Angle` directly via `set_properties` was rejected by
   the tool (bitfield/var write restriction).

3. **The big one — capture renders the *editor* world, not PIE.** Every
   `CaptureViewport` with an explicit `captureTransform` showed the static
   authored layout regardless of PIE state, and an orange wireframe box kept
   appearing (the **`GlobalPostProcess`** volume's editor visualization —
   `bUnbound=True`, so its box position is cosmetic).

### Final approach — drive the editor world directly

Since `MoveSphere` is a pure function of angle, PIE was unnecessary:

- Stopped PIE and worked in the editor world.
- Moved the (unbound) `PostProcessVolume` out of frame so its wireframe didn't
  appear (restored afterward — no visual change because it's unbound).
- For `θ = 0°…345°` in **15° steps (24 frames)**, positioned all 7 spheres at
  their exact orbit positions via the `ProgrammaticToolset`
  (`execute_tool_script`) and captured each frame. This produced perfectly
  even, jitter-free spacing with no dependence on real-time timing.
- Each `CaptureViewport` result (~3.8 MB base64 PNG) overflowed the tool output
  limit and was auto-saved to a file; decoded them locally with Python/Pillow.

### Building the GIF

- Verified the loop was seamless (consecutive frame diffs ≈ 5.0, uniform).
- Auto-cropped by projecting the known content cylinder (radius ~265,
  `z 0–700`) through the camera.
- Output: **`spiral_tornado.gif`** — 720×942, 24 frames, infinite loop,
  130 ms/frame (~3.1 s), ~3.9 MB.
- Restored the editor level to its authored state (sphere positions +
  post-process volume); did **not** save the level.

---

## 2. Move the GIF into `demo/` and reference it from the README

- Created `demo/` and moved the GIF to `demo/spiral_tornado.gif`.
- Added a **## Demo** section near the top of `README.md`:

  ```markdown
  ## Demo

  A full revolution of the sphere tornado: seven metallic rainbow spheres
  spiraling around the rainbow cube tower. This GIF was captured directly
  from the editor through MCP.

  ![Sphere tornado demo](demo/spiral_tornado.gif)
  ```

  (The user later also added Blueprint screenshots under the same section.)

---

## 3. Persisting the MCP + Terminal editor settings

**Problem:** the MCP server and Terminal startup settings kept needing to be
re-entered every session.

**Findings.** The functional settings live in the per-user file
`Saved/Config/WindowsEditor/EditorPerProjectUserSettings.ini`:

```ini
[/Script/ModelContextProtocolEngine.ModelContextProtocolSettings]
ServerPortNumber=8000
bAutoStartServer=True

[/Script/Terminal.TerminalSettings]
StartupCommands=set TERM=xterm-256color
StartupCommands=cd /d "C:\Users\rhyol\Documents\Unreal\mcp_tester"
StartupCommands=C:\Users\rhyol\.local\bin\claude.exe --dangerously-skip-permissions --resume
```

They reset because `Saved/` is **git-ignored** and Unreal only flushes editor
preferences on a **clean shutdown** (an MCP-induced crash, or wiping
`Saved/`/`Intermediate/`, loses them).

**Fix.** Created the committed project-default file
**`Config/DefaultEditorPerProjectUserSettings.ini`**. Unreal loads config in the
order *Base → Project `Config/Default*` → user `Saved/`*, so values here become
the per-session default even with `Saved/` empty:

```ini
[/Script/ModelContextProtocolEngine.ModelContextProtocolSettings]
ServerPortNumber=8000
bAutoStartServer=True

[/Script/Terminal.TerminalSettings]
+StartupCommands=set TERM=xterm-256color
+StartupCommands=cd /d "C:\Users\rhyol\Documents\Unreal\mcp_tester"
+StartupCommands=C:\Users\rhyol\.local\bin\claude.exe --dangerously-skip-permissions --resume
```

Notes given to the user:

- `+StartupCommands=` (with the `+`) is required to append array elements in a
  Default-layer ini.
- To avoid the three terminal commands being **duplicated**, delete the
  `[/Script/Terminal.TerminalSettings]` block from the `Saved/` copy once.
- Verify durability: quit, delete `Saved/`, reopen — MCP should auto-start on
  8000 and the terminal startup commands should reappear.
- **Caveats:** the file hard-codes machine-specific paths (`C:\Users\rhyol\…`)
  and auto-launches Claude with `--dangerously-skip-permissions`. If it
  shouldn't be shared, add it to `.gitignore` to keep it local-only.

---

## Artifacts produced this session

- `demo/spiral_tornado.gif` — the looping spiral-tornado animation.
- `README.md` — added the **## Demo** section.
- `Config/DefaultEditorPerProjectUserSettings.ini` — committed defaults for the
  MCP server + Terminal startup.

## Key technical takeaways

- `MoveSphere` being an absolute function of `Angle` made deterministic,
  frame-perfect capture possible without relying on PIE or real-time timing.
- `CaptureViewport` with an explicit transform renders the **editor** world, so
  the animation was reproduced by positioning actors directly in the editor
  rather than recording a live PIE session.
- Unbound `PostProcessVolume` wireframes are editor-only visualization; moving
  the actor hides the box without affecting the look.
- Editor preferences under `Saved/` are volatile (git-ignored + flushed only on
  clean shutdown); promote them to `Config/Default*.ini` to make them durable.
