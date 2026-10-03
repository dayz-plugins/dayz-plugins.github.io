---
name: vr-mod-techniques
description: >-
  How established VR mods for closed-source engines (UEVR, REFramework, R.E.A.L., F.E.A.R. VR, uuvr, vorpX, geo-11, Depth3D, VRto3D) get stable stereo and head tracking, the known causes of headset jitter, and which mechanisms transfer to a D3D11 engine with separate game and render threads such as DayZ.
game_build: not engine specific (survey of other projects; DayZ conclusions refer to 1.29.163709)
created: 2026-10-02T18:30+0200
last_edited: 2026-10-02T18:30+0200
---

# Stereo and head tracking in VR mods for engines you do not own

Sources: offline checkouts under `.references/repos/` (paths relative to it) and the public
write-ups linked inline. Written after the 2026-10-02 headset session in which DayZ-VR's
camera-basis + synthetic-mouse approach produced an "earthquake" whenever the game camera
and the HMD disagreed.

## The three rules every stable mod follows

1. **Post-correct, never pre-set.** The HMD is applied where the *renderer asks for the
   view*, after the engine produced its own camera for the frame: UEVR hooks the engine's
   stereo interface (`CalculateStereoViewOffset`, `injectors/UEVR/src/mods/vr/FFakeStereoRenderingHook.cpp:927-1040`,
   body at `:4745`) and composes `engine_rotation * hmd * eye` onto the engine's values;
   REFramework multiplies the eye transform onto the matrix the renderer requests
   (`on_camera_get_view_matrix`, `injectors/REFramework/src/mods/VR.cpp:287-310`) and fully
   replaces the projection (`:233`); R.E.A.L. applies "a supplementary view fix during
   rendering" ([gta5-real-mod](https://github.com/LukeRoss00/gta5-real-mod)); F.E.A.R. VR
   renders the LithTech camera twice with its own per-eye matrix
   ([DR-89/fear-vr](https://github.com/DR-89/fear-vr)). Nobody writes the HMD into the
   gameplay camera and lets the engine render from it. If the camera object must be
   touched, it is a save / apply / render / restore sandwich (REFramework
   `update_camera_origin` `:1531-1612` + `restore_camera`; uuvr swaps the parent rotation
   only inside `OnBeginFrameRendering`/`OnEndFrameRendering`, `injectors/uuvr/Uuvr/VrCamera/VrCamera.cs:45-62`).
2. **Submit the pose you rendered with.** The OpenXR guide is explicit: the app must hand
   back the same poses it used for rendering, otherwise "world-locked content appears to
   move in the world, proportional to the amount of head movement"
   ([frame_submission.md](https://raw.githubusercontent.com/KhronosGroup/OpenXR-Guide/main/chapters/frame_submission.md)).
   UEVR keeps a ring `pipeline_states[frame_count % N]` of `{frame_state, views}` written
   when the game thread builds matrices and consumed by the render thread at submit
   (`injectors/UEVR/src/mods/vr/runtimes/OpenXR.cpp:116-182,221-261,714`), with a
   `frame_delay_compensation` integer to bias which frame's pose the render thread picks
   (`FFakeStereoRenderingHook.hpp:591`). R.E.A.L.'s alternate-eye rendering only works
   because each submitted frame carries its exact pose so ATW reprojects the stale eye.
3. **Decouple head from body/aim.** The engine only ever sees a *flattened* yaw
   (`utility::math::flatten`, UEVR `:4890-4894`; `remove_y_component`, REFramework
   `:1588`); pitch and roll live in VR space only. Aim is pushed back into the game once
   per frame, after the engine's own rotation processing (UEVR hooks
   `APlayerCameraManager::ProcessViewRotation`, calls the original, then overwrites,
   `injectors/UEVR/src/mods/vr/IXRTrackingSystemHook.cpp:1802,836`), on the second eye
   only. Doing it per eye or before the engine's own update is a known desync source.

## Timing details that matter with a separate render thread

- Pose time: `xrLocateViews(predictedDisplayTime + period * prediction_scale)` (UEVR
  `OpenXR.cpp:261`, REFramework `OpenXR.cpp:84`); the sync stage is selectable
  EARLY / LATE / VERY_LATE = before draw, before present, after present (`UEVR/src/mods/VR.hpp:34`).
- Frame correlation: UEVR brute-forces the frame counter inside the engine's view family
  and uses it as the key between game and render thread (`FFakeStereoRenderingHook.cpp:2654-2745,3464,3677`);
  it runs code at RHI execute time by hijacking the last RHI command's vtable (`:2793-2872`).
- REFramework re-applies the camera transform *after* submitting the frame because "the
  game logic thread does not run in sync with the rendering thread … the left eye will
  jitter a lot" (`VR.cpp:1394-1406`). uuvr re-latches the pose in `Update`, `LateUpdate`
  and `OnBeforeRender`, last one wins (`Uuvr/UuvrPoseDriver.cs:41-55`).
- Engine-side camera motion is smoothed or frozen on purpose: UEVR lerps the *engine*
  rotation, never the HMD (`VR.cpp:1406-1458`) and has a `camera_freeze` latch
  (`VR.cpp:1462-1505`); F.E.A.R. VR disables camera-collision wobble and bob and rebuilds
  its reference basis only on recenter.

## Eye separation with a single engine camera

- Matrix level: `eye_transform * view_matrix` at the renderer's matrix getter gives real
  separation even when the engine has one camera (REFramework `:306`); UEVR subtracts
  `rotation * (eye_offset * world_scale)` from the view location (`:4937-4952`) and
  replaces the projection per eye with the runtime's asymmetric FOV.
- Rendering modes, best to worst (UEVR docs): native stereo (both eyes in one tick) >
  synchronized sequential (two passes, world time frozen; REFramework re-enters the
  engine's render path for the right eye and raises `MaxDeltaTime`, `VR.cpp:3063-3105`) >
  alternate frame rendering (eye per frame, "eye desyncs and usually nausea"). AFR needs:
  one head pose shared by the eye pair (UEVR `:4792-4810`, `VR.cpp:1565-1580`), motion blur,
  temporal AA and occlusion history per eye (the "ghosting fix": two view states), and the
  exact pose per submitted eye.
- Driver and shader level: geo-11 / 3DMigoto patch every vertex shader's `SV_Position`
  with separation and convergence and replay draws for both eyes
  ([helixmod](https://helixmod.blogspot.com/2022/06/announcing-new-geo-11-3d-driver.html));
  vorpX "Geometry 3D" replays draw calls with a shifted projection and its DirectVR
  scanner writes camera matrices in memory ([vorpx features](https://www.vorpx.com/features/));
  Depth3D (`tools/Depth3D/Shaders/SuperDepth3D.fx`) and vorpX "Z3D" warp a mono image by
  the depth buffer; VRto3D is a SteamVR virtual-HMD driver that repacks side-by-side
  ([VRto3D](https://github.com/oneup03/VRto3D)). None of these track the head; they are
  fallbacks for parallax only.
- Arma 3 / DayZ / Enfusion: only vorpX (Geometry 3D, TrackIR emulation, BattlEye issues)
  and no geo-11 / helixmod profile exist.

## Known jitter causes (all of them applied to DayZ-VR as of 2026-10-02)

| Cause | Where DayZ-VR had it |
| --- | --- |
| Pose written on the game thread, read by the render thread a frame later | FrameBase rotation (`+0x08..+0x20`) written per game frame, consumed by the render thread (rendering.md) |
| Submitted pose differs from the rendered pose | `xrEndFrame` uses the latest HMD pose, not the one the engine rendered |
| Two paths rotate the same camera | FrameBase write *and* synthetic mouse (closed or open loop) both rotate the view |
| Engine camera smoothing fights the head | DayZ's camera trails the mouse-driven aim; the closed-loop gain estimator diverges |
| AFR with the world advancing between eyes | alternate-eye rendering without pose pairing |

## Transfer plan for DayZ (ranked)

1. Hook the renderer's view-matrix build (candidate: projection dispatch `0x952000`, which
   "builds the cached view/projection matrices once per frame", rendering.md), call the
   original, then compose `eye * hmd` onto its output: head rotation and eye translation at
   the matrix level, the gameplay camera untouched. This also yields real eye separation,
   which the FrameBase translation never did.
2. Keep a per-frame record `{frame id, predicted display time, views}` written when the
   matrices are built and submit exactly those poses in `xrEndFrame` for that frame.
3. Feed the game only flattened yaw through the direct aim axis (input.md), once per game
   frame, after the engine's own aim update; pitch and roll stay render-only.
4. For the alternate-eye mode: one pose per eye pair, pose per submitted eye, blur off.
5. Keep the camera-freeze and engine-rotation-lerp diagnostics from UEVR to prove where
   remaining jitter comes from.
