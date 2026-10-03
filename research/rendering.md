---
name: dayz-engine-rendering
description: >-
  Frame structure, view prepare/execute/finalize, projection dispatch, camera FrameBase, FOV, HUD scale and GUI capture addresses of DayZ 1.29.163709, and the engine bugs the VR proxy works around.
game_build: DayZ 1.29.163709 (DayZ_x64.exe, PE timestamp 0x6A72FC58)
created: 2026-10-02T17:11+0200
last_edited: 2026-10-03T05:10+0200
---

# Rendering (DayZ 1.29.163709)

All addresses from `common/dayz_build_profiles.hpp` (per-build table) and
`common/dayz_build_checks.hpp` (16-byte instruction checks that guard them). The proxy
(`dxgi.dll` beside `DayZ_x64.exe`) hooks these with MinHook from inside the process.

## Frame structure

| What | RVA | Notes |
| --- | --- | --- |
| Frame function (in-world) | `0x8E77C0` | Per-frame work. Builds visibility/draw lists in the mode-0 prepare at `0x8E7967`, then calls the world render at `0x8E7AED` and `0x8E7B6B`. |
| World render | `0x8E7650` | Executes the prepared draw lists. Running it twice with a changed camera renders the **same** image: scene preparation, not this function, is the per-eye unit. |
| Projection dispatch | `0x952000` | Mode-1 per-frame dispatch; refreshes the primary FrameBase and builds the cached view/projection matrices once per frame. |
| prepareView | `0x44F5A0` | View setup; stores the prepared view pointer into the render context. |
| executeView | `0x4507A0` | Executes a view. Return value is ignored by both call sites, so a view can be skipped for one frame. |
| finalizeView | `0x4508B0` | Clears the prepared view pointer. |
| Draw call returns (GUI) | `0x25F4DD`, `0x25F5C2`, `0x25F78E` | Return addresses of the indexed/non-indexed GUI draw calls, used to attribute draws to the GUI. |

DayZ submits D3D11 on a **separate render thread** that lags the game thread by more than a
pass. Captures taken from the game thread see stale frames; immediate-context use from a
third thread crashed d3d11. The number of backbuffer-sized clears per frame is not stable
(2..18), so a clear index is not a reliable pass marker.

## Camera and FrameBase

| What | RVA / offset | Notes |
| --- | --- | --- |
| Camera manager | `0x1007CE0` (global) | Passed to the active-camera getter. |
| Active camera state getter | `0x4B6BE0` | `state = getActive(cameraManager)`. First instruction checks `+0x88 != -1`, then reads `+0x118`. |
| Camera FOV update | `0x4B7AD0` | Hooked to override the gameplay FOV in memory. |
| Profile FOV | `0x1007E00` (float global) | The loaded `.DayZProfile` fov value; the proxy writes this address only, never the file. Reference instruction at `0x5A4F24`. |
| FrameBase rotation | `+0x08..+0x20` | 3x3 basis written by the proxy for HMD yaw/pitch/roll; **honoured** by the renderer. Axes may carry different scale factors. |
| FrameBase translation | `+0x2C` | Written for eye offset and positional tracking; **ignored** by the renderer (60x scale showed no shift). Eye separation therefore never reached rendering; stereo is mono with head rotation until the renderer's real view origin is found. |

The same primary FrameBase getter is consumed by gameplay aiming and deferred rendering,
which is why head rotation written there moves both the image and the aim ray.

### Where the camera comes from each frame (1.29.163709)

- The **camera manager** (`0x1007CE0`) keeps the gameplay camera transform at
  `manager+0x50` (3x3 rotation, rows at `+0x50/+0x5C/+0x68`) and `manager+0x74` (position).
  The **primary camera object** lives at `engine(0x42638E0)+0x118`.
- The camera FOV update `0x4B7AD0` is the first thing the in-world frame function
  `0x8E77C0` calls. It compares the manager transform with the camera's `+0x08..+0x37`,
  copies it in through the camera's vtable slot 2 (`(*camera)[2](camera, manager+0x50)`),
  then calls the FrameBase refresh `0x7A0330` (call at `0x4B7DE3`).
- The FrameBase refresh `0x7A0330` builds everything derived from the camera: the view
  matrix at `camera+0x108` from the camera's vtable slot 9 getter (`0x9235A0`, which
  inverts the camera's own 3x4 `+0x08..+0x37`, translation included, through
  `0xA4D200`), its inverse, the frustum planes `+0x1B0..+0x22C` (from `+0x2C` and the
  rotation rows) and the FOV scale factors. It is called again inside the projection
  dispatch `0x952000` (call at `0x952023`) and from the inventory preview `0x5C4C79`.
- Frame order inside `0x8E77C0`: FOV update/refresh (`0x4B7AD0`) → scene passes
  `0x953240`, `0x957BF0`, `0x957100`, `0x958150`, `0x958D40` on the scene context
  `frame+0xA0` → preparation `0x85FD20` (mode-0 prepare, projection dispatch with the
  second refresh) → `0x861DE0`, `0x6DB660`, `0x6DB910/0x6DC7D0` per layer → world render
  `0x8E7650` → post/GUI. A camera translation written only in the projection dispatch
  therefore comes after the scene passes; see TASKS R5 for the experiment that writes it
  in the early refresh instead.

## HUD and GUI

| What | RVA | Notes |
| --- | --- | --- |
| HUD layout | `0x8A2280` | Hooked to read the HUD content rectangle (`GetHudContentRect`). Renderer pointer reference at `0x8A2447`. |
| GUI scale | `0x426F5CC` (float global) | Written to scale the HUD (`[hud] hud_scale`); reference instruction at `0x8A236B`. |
| GUI input handler | `0x350B60` | Window message handler; also the `WM_INPUT` consumer (see input.md). Hooked for the virtual GUI cursor. |
| Engine singleton | `0x42626D0` (global) | Root object pointer. |
| Inventory preview caller | `0x5C4CA9` | Call site of the inventory item preview preparation (`0x5C4CA4` instruction check); used to render item previews into the GUI capture. |
| Dynamic blur | `0x22EF70` | Blur effect handler; the blur amount parameter index lives at `0xFED8F8`. Zeroed per call while the proxy runs. |

Menus and the inventory render into the same backbuffer; the proxy captures them separately
and shows them on an OpenXR quad because the HUD projection does not survive the stereo path.

## Engine bugs worked around (`[patches]` ini section)

- **Null prepared view after a window drag.** The render context field `+0xA10` holds the
  prepared view pointer (set by `0x1C4440`, cleared by finalize at `0x1C57A2`) and
  `0x1B8F06` dereferences it without a check. After dragging the window the engine executed
  a view whose pointer was already null and crashed at `DayZ+0x1DDE4B`. Skipping that one
  executeView when the pointer is null drops one view for one frame and nothing else, since
  both call sites ignore the return value.

## Render-target widgets and indexed cameras (dead end, verified 2026-10-03)

The script API declares `RenderTargetWidget`, `SetWidgetWorld(widget, world, cameraIndex)`,
`SetCamera/SetCameraEx(index, ...)` and `SetCameraVerticalFOV`. An independent survey
(Codex, `/run/media/system/Data/Projects/codex/dayz-standalone-vr/docs/stereo-research.md`)
suggested the widget's native draw as a route to a second world view. Traced on 1.29.163709:

- The script bindings are live: `SetWidgetWorld` (`0x2C1750`) type-checks the widget against
  `enf::RenderTargetWidget` RTTI and stores world pointer, camera index and a flag at
  widget `+0xF8/+0x100/+0xF0` (`0x3B9EB0`); `SetCameraEx` (`0x2C07E0`) writes a 12-float
  transform into the world's camera table at `world + 0x9C + index*0xA8` (`0x10D630`).
- The widget draw virtual (`enf::RenderTargetWidget` vtable `0xC5E278` slot 6, `0x3B97D0`)
  builds the projection for the indexed camera (`0x10F5D0` from `world + 0x98 + index*0xA8`),
  sets the viewport on the renderer (`DAT_140fec3e8` slot `+0xE8`), creates or reuses the
  target texture, then calls `world->vtable[10]` (`+0x50`) with `(renderer, cameraIndex, 0)`,
  throttled by the refresh period (`+0x104`), and finally draws the texture as the widget.
- `world->vtable[10]` is `enf::BaseWorld::Render` (`0x10D270`): it refuses when the camera
  slot is disabled (`world[0x13 + index*0x15]`), refuses re-entry (`world[4]` counter) and
  when another world is mid-render, then calls `world->vtable[11]` (`+0x58`) to do the work.
- The live world is a `Landscape` (constructed at `0x9BDCF0`, vtable `0xD467C0`, global
  `DAT_1442662E0`). Its slot 11 at `0xD46818` is `0x53940`, whose bytes are `C2 00 00`
  (`ret 0`): an empty function, identical-code-folded with the other empty virtuals.
  **The indexed-camera world render is a stub in the retail client.** The in-world frame
  never uses this path either: the main loop (`0xA85090`) calls `0x8F6070`, which calls the
  frame function `0x8E77C0` directly with the globals.

Consequence: no script- or widget-driven second view exists; the second eye has to come from
re-running the frame function's scene preparation ourselves (R1). The Bohemia tracker report
DZG-926 asking for the render-target APIs to be enabled matches this finding.

## Open questions

- Where the renderer takes its view **translation** (world origin / double position) so
  that real eye separation and positional tracking become possible.
- The minimal per-eye unit inside `0x8E77C0` (scene traversal + projection + world render)
  and the state that must reset between two eyes.
- Whether an engine render-twice path exists (picture-in-picture scopes).
