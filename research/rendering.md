---
name: dayz-engine-rendering
description: >-
  Frame structure, view prepare/execute/finalize, projection dispatch, camera FrameBase, FOV, HUD scale and GUI capture addresses of DayZ 1.29.163709, the engine bugs the VR proxy works around, and the VR rendering research: an independent binary audit of this build's renderer plus two surveys of how VR mods for engines their authors do not own produce a second eye.
game_build: DayZ 1.29.163709 (DayZ_x64.exe, PE timestamp 0x6A72FC58)
created: 2026-10-02T17:11+0200
last_edited: 2026-10-04T18:30+0200
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
DZG-926 asking for the render-target APIs to be enabled matches this finding. A second,
independent Ghidra audit of the same build re-derived this chain and reached the same
conclusion; its additional evidence is in
[VR Rendering Research](#vr-rendering-research) below.

## VR Rendering Research

Condensation of three notes that stood beside this file and have been folded in here: an independent Ghidra
audit of this build's renderer, and two surveys of how VR mods for engines their authors do not own produce a
second eye — one of open engines (UEVR, REFramework, R.E.A.L., F.E.A.R. VR, uuvr, vorpX, geo-11, Depth3D,
VRto3D), written after the 2026-10-02 headset session in which DayZ-VR's camera-basis + synthetic-mouse
approach produced an "earthquake" whenever the game camera and the HMD disagreed, and one of closed
proprietary engines (2026-10-03). Neither survey is engine specific; their DayZ conclusions refer to
1.29.163709. DayZ-VR's current mode is alternate-eye.

### Independent binary audit of the same build (decompiled, Ghidra 12.1.4)

A second analysis of the installed `DayZ_x64.exe`, run deliberately without the predecessor VR project or its
research, re-derived the chain of
[Render-target widgets and indexed cameras](#render-target-widgets-and-indexed-cameras-dead-end-verified-2026-10-03)
above and reached the same conclusion: same bindings, same world vtable `0xD467C0`, same empty `ret 0` at
`0x53940` — reached without the predecessor project, so two independent analyses agree. Only the evidence that
section lacks is given here. Everything in this subsection is **decompiled**, never observed at runtime; Ghidra
names and types are inferred unless a script registration or RTTI name backs them, and the addresses are
evidence, not enabled hooks — no native gameplay stereo path is verified.

| Audited binary | Value |
| --- | --- |
| Path / digest | `/run/media/system/Data/Games/Steam/steamapps/common/DayZ/DayZ_x64.exe`, SHA-256 `6e1719275798a69d61da4f80fa57fb5f2b8d1910c95477acf1cdc73da9af3129` — **verified** against the installed binary on 2026-10-04. Lowercase on purpose: a digest to string-compare against `sha256sum`, not an address. |
| Build | PE `TimeDateStamp` `0x6A72FC58` = 2026-08-05 09:03:20 UTC (11:03:20 CEST), read back from the PE header on 2026-10-04; the source note's "11:03:20 UTC" mislabelled the zone. Matching this stamp is what establishes that both analyses examined the build in this file's frontmatter. Embedded source branch `_continuous_branches_stable_0129`. |
| Analysis | Ghidra 12.1.4, x86-64 Windows compiler specification; preferred image base `0x140000000`. |

| Chain evidence the render-target section lacks | RVA / field | Note |
| --- | --- | --- |
| Script name string `SetWidgetWorld` | `0xC41778` | registered at `0x2AB9DB` to binding `0x2C1750`; the binding requires a non-null world |
| Native world accessor | `0x7B8110` | returns the global at `0x42662E0` (the `DAT_1442662E0` of the section above), ignoring its input object; the world constructor result is stored there at `0x44031F`/`0x44032C`, and `0x10DA60` then republishes the world to `0xF900F0` |
| Widget scene call sites | `0x3B9A50`, `0x3B9CB3` | both dispatch world vtable `+0x50` |
| Base-class constructor | `0x109600` | called by the concrete world construction `0x9BDCF0`; initialises 32 camera records |
| World vtable slot `+0x50` | `0xD46810` | holds `0x10D270`, whose onward call to slot `+0x58` sits at `0x10D3A6`. (Slot `+0x58` at `0xD46818` and its `C2 00 00` body are already recorded above.) A widget allocation or camera move succeeding therefore proves nothing about gameplay stereo, and the result is conclusive only for this concrete class: other world switches exist, so read the runtime object's own vtable for an unusual scene or build. |
| Widget binding `0x3B9EB0` writes `+0xF0` | widget `+0xF0` | **The two analyses disagree**: the section above reads this field as a flag, the independent audit as the renderer pointer (`+0xF8` world and `+0x100` camera id are agreed). Unresolved; do not rely on either reading. |
| Slot `+0x58` call signature | — | `world->vtable[0x58/8](world, renderer, cameraId, 0, flags \| 1)` — a fifth argument the section above does not record |

| Indexed camera record, `world + 0x98 + index*0xA8` (widget path only, 32 records) | Use |
| --- | --- |
| `+0x00` | camera type; zero makes `0x10D270` return without dispatching |
| `+0x04..+0x30` | twelve 32-bit transform components (`+0x04` is the field the `world + 0x9C + index*0xA8` write above lands on); `+0x28..+0x30` position, written separately by `SetCamera` |
| `+0x34` / `+0x3C` / `+0x40..+0x4C` | far plane via `0x10D800` / near plane via `0x10D830` / four projection inputs copied by `0x10D7A0` |
| `+0x50` / `+0x54` / `+0x58` | extra aspect-projection adjustment / selects the symmetric projection branch, with `0x10F5D0` using asymmetric math when it is zero — useful engine evidence that does not make the widget scene dispatch functional / projection-related invalidation flag |

`SetCamera` (`0x2C07C0` → `0x10D6E0`) derives orientation and copies position. These offsets must not be
conflated with the active gameplay camera at `engine(0x42638E0)+0x118`
([Where the camera comes from each frame](#where-the-camera-comes-from-each-frame-129163709) above), for which
two script registrations — a stronger identity than an inferred name — pin down the read path:

| Binding | Registration (name string) | Read impl | Reads |
| --- | --- | --- | --- |
| `GetCurrentCameraPosition` | `0x5BA539` (`0xCB15C8`) | `0x5B2E10` | three floats at `camera+0x2C` |
| `GetCurrentCameraDirection` | `0x5BA553` (`0xCB15E8`) | `0x5B2DE0` | three floats at `camera+0x20` |

The audit claims only the two field offsets; that `camera+0x2C` is the same field as the FrameBase translation
and `camera+0x20` sits inside the FrameBase basis of [Camera and FrameBase](#camera-and-framebase) is this
file's inference from the offsets agreeing, not the audit's claim. `engine+0x118` and those two fields are all
the bindings establish — matrices, ownership, frame lifetime and culling dependencies still need tracing before
the camera is modified. Reproduce from the independent Ghidra project
`/run/media/system/Data/Projects/codex/dayz-standalone-vr/build/ghidra-render/DayZRender.gpr` (the same codex
tree whose `docs/stereo-research.md` is cited in the render-target section above), reused read-only with
`-process DayZ_x64.exe -noanalysis -readOnly` so extraction patches neither binary nor database, inside
`build-box` at 2 CPUs / 8 GiB heap / nice 10, via that project's `scripts/ghidra/`:
`analyze-renderer.sh functions build/ghidra-render/widget-evidence.c 0x2C1750 0x7B8110 0x3B9EB0 0x3B97D0
0x10D270 0x9BDCF0 0x53940`. The output records callers, direct callees, decompiled functions and entry
instruction bytes; database and full disassembly stay in ignored build output.

### How other projects produce a second eye

Both surveys read offline checkouts under the predecessor project's `.references/repos/` (proprietary ones
under `.references/repos/proprietary/`, indexed by `.references/repositories.tsv`) — not part of this
repository — plus the write-ups linked inline. The proprietary repos were found through the Flat2VR Discord
archive on the NAS (`/var/mnt/nas/projects/vr mods/flat2vr`, `archive.sqlite` full-text search) and each was
read in full by a separate analysis pass on 2026-10-03; all `file:line` pointers are repo-relative.
**No Bohemia-engine (Real Virtuality / Enfusion) VR mod with public source exists in that archive** — the Arma
Reforger and "Dayz sa vr" threads are requests only.

#### Shared conclusions and rules

| Conclusion | Citations |
| --- | --- |
| **Post-correct, never pre-set.** The pose is applied where the renderer *asks for the view*, after the engine produced its own camera for the frame — which may be the camera object itself, written idempotently per frame (Wildlands' `Camera+0x000` at the projection-builder hook, `out = H * base` from a base captured once per frame) or the view out-parameters (BioShock's `CalcView`). What nobody does is pre-set the HMD into the gameplay camera and let the engine evolve it; where the camera object is touched outside the renderer's request it is a save / apply / render / restore sandwich. | UEVR hooks `CalculateStereoViewOffset` (`injectors/UEVR/src/mods/vr/FFakeStereoRenderingHook.cpp:927-1040`, body at `:4745`) and composes `engine_rotation * hmd * eye` onto the engine's values; REFramework multiplies the eye transform onto the matrix the renderer requests (`on_camera_get_view_matrix`, `injectors/REFramework/src/mods/VR.cpp:287-310`) and fully replaces the projection (`:233`); R.E.A.L. applies "a supplementary view fix during rendering" ([gta5-real-mod](https://github.com/LukeRoss00/gta5-real-mod)). Sandwich: REFramework `update_camera_origin` `:1531-1612` + `restore_camera`; uuvr swaps the parent rotation only inside `OnBeginFrameRendering`/`OnEndFrameRendering` (`injectors/uuvr/Uuvr/VrCamera/VrCamera.cs:45-62`). vrframework prefers out-parameter rewrite over a constant-buffer patch at known offsets. |
| **Submit the pose you rendered with**, or "world-locked content appears to move in the world, proportional to the amount of head movement" (OpenXR guide [frame_submission.md](https://raw.githubusercontent.com/KhronosGroup/OpenXR-Guide/main/chapters/frame_submission.md)). | UEVR keeps a ring `pipeline_states[frame_count % N]` of `{frame_state, views}`, written when the game thread builds matrices and consumed by the render thread at submit (`injectors/UEVR/src/mods/vr/runtimes/OpenXR.cpp:116-182,221-261,714`), plus a `frame_delay_compensation` integer biasing which frame's pose the render thread picks (`FFakeStereoRenderingHook.hpp:591`). R.E.A.L.'s alternate-eye rendering only works because each submitted frame carries its exact pose, so ATW reprojects the stale eye. BioShock submits the **latched** pose, per-eye position = latched centre ± half IPD along the latched right vector (`XRSession.cpp:795-852`); a fresh pose at submit baked about 3 degrees of yaw at 200 deg/s, and Wildlands saw a 36 Hz monocular shimmer on head turns with the present-time pose (`VRMirror.cpp:3879-3900`). Racer calls `xrLocateViews(predictedDisplayTime)` once per frame before drawing and those cached views are what `xrEndFrame` gets. |
| **Pair eye with frame by a queue, not by parity.** Wildlands and BioShock both reject frame parity for a queue of eye tags written by the thread that composed the camera and popped exactly one per Present, so no lag constant is needed; BioShock's health metric is "queue depth min == max == 1". | Wildlands: ring, never parity ("build-to-present depth flaps 0..1"), the tag carrying the exact HMD quaternion the frame was composed with (`HeadPose.h:271-317`). BioShock: 64-slot FIFO of eye tags, game thread → Present (`CameraHook.cpp:4509,4561`). Counter-examples: vrframework uses parity with a drift guard that drops one present to re-phase; UnrealVR an explicit one-frame skew with a `Tick` detour flipping the eye; Detroit VR a `declared_pose_frame_lag` setting plus the measured pose, present-anchored, with depth submit. |
| **A closed engine's whole scene render can be re-entered twice.** Four projects re-ran theirs, so vrframework's warning that closed render paths "cannot be safely re-entered" is a default, not a law. This *converges* with — it does not confirm — the [Frame structure](#frame-structure) finding that for DayZ the mode-0 scene preparation and not the world render `0x8E7650` is the per-eye unit: those projects re-entered their own engines, not this one. | BladeVR and Racer both got correct per-eye portals, shadows, reflections and culling by calling the engine's whole scene render twice rather than duplicating draws (`SceneCullingRootHook.cpp:394-447`; `renderer_hook.cpp:3463-3484`); BladeVR's 1.2 note: draw duplication alone left "black slivers" at doorway edges, the full re-run fixed it. REFramework re-enters the engine's render path for the right eye and raises `MaxDeltaTime` (`VR.cpp:3063-3105`); F.E.A.R. VR renders the LithTech camera twice with its own per-eye matrix ([DR-89/fear-vr](https://github.com/DR-89/fear-vr)). DayZ's own guard is the `BaseWorld::Render` re-entry counter (above), which the frame-function path does not use. |
| **Decouple head from body/aim** — the third of the open-engine survey's "three rules every stable mod follows", not a cross-survey convergence: the engine sees a *flattened* yaw; pitch and roll live in VR space only; aim is pushed back once per frame after the engine's own rotation processing and on the second eye only — per eye, or before the engine's own update, is a known desync source. | `utility::math::flatten` (UEVR `:4890-4894`), `remove_y_component` (REFramework `:1588`); UEVR hooks `APlayerCameraManager::ProcessViewRotation`, calls the original, then overwrites (`injectors/UEVR/src/mods/vr/IXRTrackingSystemHook.cpp:1802,836`). |

#### Proprietary-engine mods, per project

| Mod (engine, API) | Second eye | Where the pose is written |
| --- | --- | --- |
| BladeVR (Blade of Darkness remaster, D3D11 via bgfx, CPU T&L, single thread) | engine world render called twice per frame with the camera shifted ±IPD/2; batches routed to cloned bgfx views with half-screen rects, draws issued once per eye with a per-eye projection constant buffer | the engine's global view matrix, hooked after its single writer, on the game thread, last call of the frame only; pose stored at the matrix write and attached to Submit; no eye/frame pairing needed (both eyes in one frame) |
| Ghost Recon Wildlands VR (AnvilNext, D3D11) | alternate eye: eye toggled once per built frame, offset along the camera's right row | `Camera+0x000` 4x4 at the projection-builder hook on the engine thread, idempotent per frame (`out = H * base` from a base captured once per frame) |
| BioShock Remastered VR (UE3 fork, D3D11) | alternate eye: offset along the rotator's right vector at `PlayerCalcView` | the `CalcView` out-parameters (location, rotation), leader call site auto-detected; latched-pose seqlock from the game thread |
| aliasIsolation (Alien: Isolation, D3D11; no VR, the base MotherVR built on) | none; the complete recipe for patching view/projection instead (below) | n/a |
| Star Wars Racer PCVR (SW_RACER_RE, OpenGL) | synchronous: the scene-render function is replaced by a loop running the original body once per eye | view = located per-eye pose post-multiplied into the game view; projection = runtime frustum, engine near plane kept |
| UnrealVR (UE4, D3D11) | alternate eye; projection out-parameter overwritten with the runtime's asymmetric frustum | own CameraActor as view target, driven through engine calls; one pose per eye pair — `xrWaitFrame`/`xrBeginFrame`/`xrLocateViews` on the left Present, `xrEndFrame` on the right |
| vrframework (guide + scaffold; RE Engine, Creation 2, Anvil, FH5) | recommends alternate frame rendering | pose sampled at the last edge before the engine bakes view/projection; three counters (engine/render/presenter) |
| Detroit VR (Vulkan) | "native stereo" ladder: multiview → run the render graph twice → own descriptors → patched SPIR-V with a per-eye correction matrix in the pass constant buffer | engine |

#### Timing, pacing and hooking with a separate render thread

| Topic | Detail |
| --- | --- |
| Pose time, sync stage | `xrLocateViews(predictedDisplayTime + period * prediction_scale)` (UEVR `OpenXR.cpp:261`, REFramework `OpenXR.cpp:84`); sync stage selectable EARLY / LATE / VERY_LATE = before draw / before present / after present (`UEVR/src/mods/VR.hpp:34`). |
| Frame correlation | UEVR brute-forces the frame counter inside the engine's view family and uses it as the game↔render-thread key (`FFakeStereoRenderingHook.cpp:2654-2745,3464,3677`); it runs code at RHI execute time by hijacking the last RHI command's vtable (`:2793-2872`). |
| Re-latch late, re-apply after submit | REFramework re-applies the camera transform *after* submitting the frame because "the game logic thread does not run in sync with the rendering thread … the left eye will jitter a lot" (`VR.cpp:1394-1406`). uuvr re-latches the pose in `Update`, `LateUpdate` and `OnBeforeRender`, last one wins (`Uuvr/UuvrPoseDriver.cs:41-55`). |
| Smooth the engine, never the HMD | UEVR lerps the *engine* rotation only (`VR.cpp:1406-1458`) and has a `camera_freeze` latch (`VR.cpp:1462-1505`); F.E.A.R. VR disables camera-collision wobble and bob and rebuilds its reference basis only on recenter. |
| Pacing | Keep `xrWaitFrame` the only clock: Racer and Detroit force desktop vsync off; Wildlands moved `xrWaitFrame` to a worker with a timeout because blocking it on the render thread froze the game; UnrealVR skips the real `Present` on the right eye to halve desktop presents. |
| Hooking | Hook the context from the first real Present, not the first device, because decoy swapchains exist (BladeVR `Dx11Hook.cpp:97-101`); never create hooks inside the game's `CreateSwapChain`; verify prologue bytes before hooking by offset so a game patch disables VR instead of crashing (BladeVR `HookGameFunction`; Wildlands' per-build RVA table keyed by PE timestamp and image size, the role `common/dayz_build_profiles.hpp` plays here, guarded by the byte checks in `common/dayz_build_checks.hpp`). |
| Honesty | Runtime A/B switches for every hook group (aliasIsolation Ctrl+Del/Ins) and an invariants file of settled and falsified hypotheses (BioShock `docs/INVARIANTS.md`) kept those projects honest; TASKS.md plays that role here. |

#### Eye separation and mode ranking

UEVR's docs rank native stereo (both eyes in one engine tick) > synchronized sequential (two passes, world time
frozen) > alternate frame rendering (eye per frame, "eye desyncs and usually nausea").

| Level | Mechanism |
| --- | --- |
| Matrix, any mode | `eye_transform * view_matrix` at the renderer's matrix getter gives real separation even when the engine has one camera (REFramework `:306`); UEVR subtracts `rotation * (eye_offset * world_scale)` from the view location (`:4937-4952`) and replaces the projection per eye with the runtime's asymmetric FOV. |
| Scene re-run prerequisites | Every per-frame cache keyed to a frame counter must see a new frame on pass 2 (BladeVR increments the engine's counter); the culling camera must be widened or re-run per eye (Racer widens the FOV for culling only, `renderer_hook.cpp:2427-2500`); once-per-frame side effects (weather particles, random light flicker, timers) must be gated to the last pass (`vr_last_eye_pass()`; BladeVR suppresses `rand()` light intensity on pass 2); depth and colour must be cleared per pass. A partial, other-engine answer to the open question below about state that must reset between two eyes; the DayZ-specific list is unverified. |
| Routing the second pass | BladeVR clones the renderer's view objects with a half-screen rect (bgfx specific); Racer blits the default framebuffer into a per-eye texture after each pass. Our `double_capture_clear` backbuffer capture after pass 1 is the equivalent and already works — the blocker was the identical image, not the capture. |
| Pair coherency (AFR) | One head pose shared by the eye pair (UEVR `:4792-4810`, `VR.cpp:1565-1580`); BioShock's **pair lock**: eye 0 snapshots camera rotation and location, eye 1 is forced to the same snapshot (`CameraHook.cpp:3311-3334`). Motion blur off, temporal AA and occlusion history per eye (the "ghosting fix": two view states), exact pose per submitted eye; both projects forbid TAA while alternating eyes. Wildlands shares **one head orientation for both views** because the per-eye orientations from `xrLocateViews` carry the headset's cant, which the content does not have (`VRMirror.cpp:3870-3878`). |
| Constant-buffer patch (fallback) | aliasIsolation's full recipe: identify passes by DXBC MD5 at `CreateVertexShader`/`CreatePixelShader`, capture the constant buffer by slot at that pass, record the mapped pointer in `Map`, rewrite the matrices in `Unmap` before the real call (`shaderHooks.cpp:77-184`, `taa.cpp:224-419`); keep shadow passes out by checking that the camera position in the buffer matches the one reconstructed from the view matrix (shadow passes do not update it), keep planar reflections out with an identity secondary projection, only touch buffers whose render target and viewport are screen-sized, and null every cached pointer on `ResizeBuffers`. BladeVR identifies the projection buffer by **shape** (row-vector perspective: `m[11]==1`, `m[15]==0`, zeros elsewhere; `ProjectionHook.cpp:55`) and binds a per-eye copy per draw, with a separate no-offset pair for the sky; vrframework's FH5 adapter does the same for D3D12 with a per-slot hash so a buffer is not shifted twice per frame. Caveat for DayZ: parallax for geometry only — screen-space passes (AO, bloom, blur, SSR) and the visibility lists stay mono, and Blade tells users to disable AA, bloom, motion blur and AO. Diagnostic or stepping stone, never a shipping mode. |
| Parallax-only fallbacks, **no head tracking** | geo-11 / 3DMigoto patch every vertex shader's `SV_Position` with separation and convergence and replay draws for both eyes ([helixmod](https://helixmod.blogspot.com/2022/06/announcing-new-geo-11-3d-driver.html)); vorpX "Geometry 3D" replays draw calls with a shifted projection and its DirectVR scanner writes camera matrices in memory ([vorpx features](https://www.vorpx.com/features/)); Depth3D (`tools/Depth3D/Shaders/SuperDepth3D.fx`) and vorpX "Z3D" warp a mono image by the depth buffer; VRto3D is a SteamVR virtual-HMD driver that repacks side-by-side ([VRto3D](https://github.com/oneup03/VRto3D)). For Arma 3 / DayZ / Enfusion only vorpX exists (Geometry 3D, TrackIR emulation, BattlEye issues) — no geo-11 / helixmod profile. |

#### DayZ-specific caveats, and the jitter causes DayZ-VR had on 2026-10-02

| Item | Detail |
| --- | --- |
| World-delta clamp does **not** transfer | BioShock's clamp (frame delta returns 0 on the right eye, delta plus carry on the left, so the world advances once per pair, `CameraHook.cpp:1725-1800`) is single-player only. DayZ's simulation is server-authoritative, so freezing or halving the client's frame delta desyncs interpolation, animation and the network tick. What transfers is the pair lock alone: one HMD sample and one eye-pair centre for both frames of a pair (we freeze per frame today), world advancing normally, compositor reprojection covering the difference. Both eyes showing the same instant is only possible with synchronous stereo. |
| Capture pollution | Wildlands reading the camera back read the mod's own write and compounded into discrete 1x/2x offsets ("three image positions per eye"); a candidate base within 0.10 m of the last write is rejected (`CameraProbe.cpp:1839-1888`). Our `ApplyHmdRotationToCamera` reads the camera manager each frame and composes; verify that base is the engine's and not our own previous write — the FrameBase refresh copies manager to camera, so the manager should be clean, but the prepare-time write must not feed back. |
| Jitter: pose written on the game thread, read by the render thread a frame later | FrameBase rotation (`+0x08..+0x20`) written per game frame, consumed by the render thread ([Camera and FrameBase](#camera-and-framebase) above) |
| Jitter: submitted pose differs from the rendered pose | `xrEndFrame` used the latest HMD pose, not the one the engine rendered |
| Jitter: two paths rotate the same camera | FrameBase write *and* synthetic mouse (closed or open loop) both rotated the view |
| Jitter: engine camera smoothing fights the head | DayZ's camera trails the mouse-driven aim; the closed-loop gain estimator diverges |
| Jitter: AFR with the world advancing between eyes | alternate-eye rendering without pose pairing |

### Transfer plan and queued experiments (merged, ranked by expected payoff)

1. **Re-run the mode-0 scene preparation** for the second eye with the engine frame counter advanced
   and culling widened, capturing after pass 1 as today (R1; prerequisites above). Highest payoff, and
   the route both surveys endorse.
2. **Hook the renderer's view-matrix build** — candidate: the projection dispatch `0x952000` from
   [Frame structure](#frame-structure) above — call the original, then compose `eye * hmd` onto its output:
   head rotation and eye translation at the matrix level, gameplay camera untouched. This also yields the
   eye separation the ignored FrameBase translation `+0x2C` never did.
3. **Submit the rendered pose** (`submit_rendered_pose=true`). The per-frame records already exist and are
   the mechanism three of these projects converged on; what is new is logging ring occupancy and switching
   from the `frame_lag`-indexed ring to pop-per-Present eye tags once the measured depth is stable.
4. **Layer orientation from the head pose**, per-eye position only — check first whether ours uses the
   runtime's per-eye view poses.
5. **Pair lock for alternate-eye mode**: one HMD sample and one eye-pair centre per pair, pose per submitted
   eye, world time unchanged, motion blur off ([HUD and GUI](#hud-and-gui): dynamic blur is already zeroed
   per call).
6. **Flattened yaw only** into the game, through the direct aim axis ([input.md](input.md)), once per game
   frame, after the engine's own aim update; pitch and roll stay render-only.
7. Verify the HMD rotation composes on the engine's camera base, not on our own previous write, and keep
   temporal AA off in the deployed ini for the headset test.
8. Only if 1 stays blocked: **constant-buffer patch** of the view/projection at the main pass, as a
   diagnostic.
9. Keep UEVR's camera-freeze and engine-rotation-lerp diagnostics to prove where remaining jitter
   comes from.

## Open questions

- Where the renderer takes its view **translation** (world origin / double position) so
  that real eye separation and positional tracking become possible. Candidate answer:
  compose the eye offset onto the projection dispatch's matrix output (VR Rendering
  Research, transfer plan item 2).
- The minimal per-eye unit inside `0x8E77C0` (scene traversal + projection + world render)
  and the state that must reset between two eyes — other engines' prerequisites are listed
  under *Scene re-run prerequisites* above; the DayZ-specific list is still unverified.
- Whether an engine render-twice path exists (picture-in-picture scopes).
