---
name: proprietary-engine-vr-mods
description: >-
  Source-level survey of VR mods for closed proprietary engines (BladeVR, Ghost Recon Wildlands VR, BioShock Remastered VR, aliasIsolation/MotherVR lineage, Star Wars Racer PCVR, UnrealVR, vrframework, Detroit VR): how each produces the second eye, where the head pose is written, how the submitted pose is kept equal to the rendered one, and which pieces transfer to DayZ.
game_build: not engine specific (survey; DayZ conclusions refer to 1.29.163709)
created: 2026-10-03T06:40+0200
last_edited: 2026-10-03T06:40+0200
---

# VR mods for proprietary engines: rendering techniques

Sources: the Flat2VR Discord archive on the NAS (`/var/mnt/nas/projects/vr mods/flat2vr`,
`archive.sqlite` full-text search) pointed at the repositories, which are cloned under
`.references/repos/proprietary/` (listed in `.references/repositories.tsv`). Each repo was read
in full by a separate analysis pass on 2026-10-03; file:line pointers are relative to the repo.
Complements [vr-mod-techniques.md](vr-mod-techniques.md) (UEVR, REFramework, R.E.A.L., ...).

No Bohemia-engine (Real Virtuality / Enfusion) VR mod with public source exists in the archive;
the Arma Reforger and "Dayz sa vr" threads are requests only.

## One table

| Mod (engine, API) | Second eye | Where the pose is written | Submitted pose = rendered pose | Pairing eye with frame |
| --- | --- | --- | --- | --- |
| BladeVR (Blade of Darkness remaster, D3D11 via bgfx, CPU T&L, single thread) | engine world render called twice per frame with the camera shifted ±IPD/2, batches routed to cloned bgfx views with half-screen rects, draws issued once per eye with a per-eye projection constant buffer | the engine's global view matrix, hooked after its single writer, on the game thread, last call of the frame only | pose stored at the matrix write, attached to Submit | n/a (both eyes in one frame) |
| Ghost Recon Wildlands VR (AnvilNext, D3D11) | alternate eye: eye toggle once per built frame, offset along the camera's right row | `Camera+0x000` 4x4 at the projection-builder hook on the engine thread, idempotent per frame (`out = H * base` from a base captured once per frame) | ring of eye tags pushed per built frame, popped per Present; tag carries the exact HMD quaternion the frame was composed with | ring, never parity ("build-to-present depth flaps 0..1") |
| BioShock Remastered VR (UE3 fork, D3D11) | alternate eye: offset along the rotator's right vector at `PlayerCalcView` | the `CalcView` out-parameters (location, rotation), leader call site auto-detected | latched pose seqlock from the game thread; per-eye position = latched centre ± half IPD along the latched right vector | 64-slot FIFO of eye tags, game thread → Present |
| aliasIsolation (Alien: Isolation, D3D11; no VR, the base MotherVR built on) | none; shows how to patch view/projection: capture the engine constant buffers by slot at a hash-identified pass, rewrite matrices in `Unmap` before the real call | n/a | n/a | n/a |
| Star Wars Racer PCVR (SW_RACER_RE, OpenGL) | synchronous: the scene-render function is replaced by a loop that runs the original body once per eye | view = located per-eye pose post-multiplied into the game view; projection = runtime frustum, engine near kept | `xrLocateViews(predictedDisplayTime)` once per frame before drawing; the cached views are what `xrEndFrame` gets | n/a |
| UnrealVR (UE4, D3D11) | alternate eye; projection out-parameter overwritten with the runtime's asymmetric frustum | own CameraActor as view target, driven through engine calls | one pose per eye pair: `xrWaitFrame/xrBeginFrame/xrLocateViews` on the left Present, `xrEndFrame` on the right | explicit one-frame skew; `Tick` detour flips the eye |
| vrframework (guide + scaffold; RE Engine, Creation 2, Anvil, FH5) | recommends alternate frame rendering because closed render paths "cannot be safely re-entered" | out-parameter rewrite (preferred) or constant-buffer patch at known offsets | pose sampled at the last edge before the engine bakes view/projection; three counters (engine/render/presenter) | parity with a drift guard that drops one present to re-phase |
| Detroit VR (Vulkan) | "native stereo" ladder: multiview → run the render graph twice → own descriptors → patched SPIR-V with a per-eye correction matrix in the pass constant buffer | engine | `declared_pose_frame_lag`, present-anchored pose, depth submit | frame lag setting plus measured pose |

## What transfers to DayZ, in order of expected payoff

### A. Pose pairing and submission (cheap, directly comparable to the current design)

Three independent projects converged on the same mechanism we implemented as frame records:
- Wildlands and BioShock both reject frame parity and use a **queue of eye tags** written by the
  thread that composed the camera and consumed at Present (`HeadPose.h:271-317`;
  `CameraHook.cpp:4509,4561`). Ours is a ring indexed by `frame_lag`; their queue needs no lag
  constant because it pops exactly one tag per Present. BioShock's health metric is "queue depth
  min == max == 1". Worth adopting: log the ring occupancy and switch to pop-per-Present if the
  measured depth is stable.
- BioShock submits the **latched** pose with per-eye position = latched centre ± half IPD along
  the latched right vector (`XRSession.cpp:795-852`); a fresh pose at submit gave about 3 degrees
  of baked yaw at 200 deg/s. Wildlands saw a 36 Hz monocular shimmer on head turns with the
  present-time pose (`VRMirror.cpp:3879-3900`). This is `submit_rendered_pose=true` for us.
- Wildlands shares **one head orientation for both views**: the per-eye orientations from
  `xrLocateViews` carry the headset's cant, which the content does not have
  (`VRMirror.cpp:3870-3878`). Check whether our per-eye view poses from the runtime are used
  for the layer orientation; if so, use the head orientation plus per-eye position only.

### B. Alternate-eye coherency (cheap, our current mode)

- BioShock's **pair lock**: eye 0 snapshots camera rotation and location, eye 1 is forced to the
  same snapshot (`CameraHook.cpp:3311-3334`). Plus a **world-delta clamp**: the frame-delta
  function returns 0 on the right eye and delta plus carry on the left, so the world advances
  once per pair (`:1725-1800`). The delta clamp is **single-player only**: DayZ's simulation is
  server-authoritative, so freezing or halving the client's frame delta desyncs interpolation,
  animation and the network tick. What transfers is the pair lock alone: one HMD sample and one
  eye-pair centre for both frames of a pair (we freeze per frame today), with the world
  advancing normally and the compositor's reprojection covering the difference. Both eyes
  showing the same instant is only possible with synchronous stereo (C).
- Wildlands' **capture-pollution guard**: reading the camera back read the mod's own write and
  compounded into discrete 1x/2x offsets ("three image positions per eye"); a candidate base
  within 0.10 m of the last write is rejected (`CameraProbe.cpp:1839-1888`). Our
  ApplyHmdRotationToCamera reads the camera manager each frame and composes; verify that the
  base we read is the engine's, not our own previous write (the FrameBase refresh copies manager
  to camera, so the manager should be clean, but the prepare-time write must not feed back).
- Both forbid TAA while alternating; our deployed ini should keep temporal AA off for the
  headset test.

### C. Synchronous stereo by re-running the engine's own render (the R1 route)

- BladeVR and Racer both got correct per-eye portals, shadows, reflections and culling by
  **calling the engine's whole scene render twice** rather than duplicating draws
  (`SceneCullingRootHook.cpp:394-447`; `renderer_hook.cpp:3463-3484`). BladeVR's 1.2 note:
  draw duplication alone left "black slivers" at doorway edges, the full re-run fixed it. This is
  exactly our R1 finding that the mode-0 scene preparation, not the world render, is the per-eye
  unit.
- Prerequisites they list: every per-frame cache keyed to a frame counter must see a new frame
  on pass 2 (BladeVR increments the engine's counter); the engine's culling camera must be
  widened or re-run per eye (Racer widens the FOV for culling only, `renderer_hook.cpp:2427-2500`);
  once-per-frame side effects (weather particles, random light flicker, timers) must be gated
  to the last pass (`vr_last_eye_pass()`; BladeVR suppresses `rand()` light intensity on pass 2);
  depth and colour must be cleared per pass.
- Routing the second pass to its own target: BladeVR clones the renderer's view objects with a
  half-screen rect (bgfx specific); Racer blits the default framebuffer into a per-eye texture
  after each pass. For DayZ, the simplest equivalent is our existing capture of the backbuffer
  after pass 1 (`double_capture_clear`), which already works; the blocker was the identical image,
  not the capture.
- vrframework's warning that closed render paths cannot be re-entered is a default, not a law:
  both Blade and Racer re-enter theirs. DayZ's own guard is the re-entry counter in
  `BaseWorld::Render` (rendering.md), which the frame function path does not use.

### D. Matrix-level stereo without re-running the engine (fallback if C stays blocked)

- aliasIsolation is the complete recipe for **patching view/projection constant buffers** in a
  D3D11 engine: identify passes by DXBC MD5 at `CreateVertexShader/CreatePixelShader`, capture
  the constant buffer by slot at that pass, record the mapped pointer in `Map`, rewrite the
  matrices in `Unmap` before the real call (`shaderHooks.cpp:77-184`, `taa.cpp:224-419`); keep
  shadow passes out by checking that the camera position in the buffer matches the position
  reconstructed from the view matrix (shadow passes do not update it), keep planar reflections
  out by an identity secondary projection, and only touch buffers whose render target and
  viewport are screen-sized. Null every cached pointer on `ResizeBuffers`.
- BladeVR identifies the projection buffer by **shape** (row-vector perspective: `m[11]==1`,
  `m[15]==0`, zeros elsewhere; `ProjectionHook.cpp:55`) and binds a per-eye copy per draw, with a
  separate no-offset pair for the sky. vrframework's FH5 adapter does the same for D3D12 with a
  per-slot hash so a buffer is not shifted twice per frame.
- Caveat for DayZ: this gives parallax for geometry only. Anything computed in screen space
  (AO, bloom, blur, SSR) and the visibility lists are still mono; Blade tells users to disable
  AA, bloom, motion blur and AO. Use only as a diagnostic or stepping stone.

### E. Frame pacing and lifecycle (hygiene)

- Keep `xrWaitFrame` the only clock: Racer and Detroit force desktop vsync off; Wildlands moved
  `xrWaitFrame` to a worker with a timeout because blocking it on the render thread froze the
  game. UnrealVR skips the real `Present` on the right eye to halve desktop presents.
- Hook the context from the first real Present, not the first device: decoy swapchains exist
  (BladeVR `Dx11Hook.cpp:97-101`); never create hooks inside the game's `CreateSwapChain`.
- Verify prologue bytes before hooking by offset so a game patch disables VR instead of
  crashing (BladeVR `HookGameFunction`; Wildlands' per-build RVA table keyed by PE timestamp and
  image size, which is what `dayz_build_checks.hpp` does).
- Runtime A/B switches for every hook group (aliasIsolation Ctrl+Del/Ins) and an invariants file
  listing settled and falsified hypotheses (BioShock `docs/INVARIANTS.md`) kept their projects
  honest; TASKS.md plays that role here.

## Experiments queued from this survey

1. Ring occupancy log and pop-per-Present eye tags (A).
2. Layer orientation from the head pose, per-eye position only (A).
3. Pair lock (one pose per eye pair, no world-time change) for alternate-eye mode (B).
4. Re-run the mode-0 scene preparation for the second eye with the frame counter advanced and
   culling widened (C), capturing after pass 1 as today.
5. Only if 4 stays blocked: constant-buffer patch of the view/projection at the main pass (D).
