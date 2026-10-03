---
name: dayz-engine-research
description: >-
  Index of the DayZ (Enfusion) engine reverse-engineering notes: which file covers rendering, input, scripting and the Ghidra tooling, plus the conventions used in all of them.
game_build: DayZ 1.29.163709 (DayZ_x64.exe, PE timestamp 0x6A72FC58)
created: 2026-10-02T17:11+0200
last_edited: 2026-10-03T06:40+0200
---

# DayZ engine research notes

Condensed reverse-engineering findings about the DayZ (Enfusion) client, written for other
projects, modders and agents. Everything here was established on **DayZ 1.29.163709**
(`DayZ_x64.exe`, PE timestamp `0x6A72FC58`, image size `0x04406000`) unless a build is named;
addresses are RVAs relative to the image base (`DayZ+0x...`). Other builds shift every address,
so verify with the byte checks in `common/dayz_build_checks.hpp` or re-run the Ghidra queries.

| File | Topic |
| --- | --- |
| [rendering.md](rendering.md) | Frame structure, view prepare/execute/finalize, projection dispatch, camera (FrameBase), FOV, HUD scale, GUI capture, engine bugs the proxy works around |
| [input.md](input.md) | Input system: raw input and XInput device layer, the action registry (`UAInput` records), HumanInputController action tables, focus gating, what can be written from outside |
| [scripting.md](scripting.md) | Enforce Script facts that matter for native code: native binding tables, what the client can and cannot override without a server mod, file bridge |
| [vr-mod-techniques.md](vr-mod-techniques.md) | How UEVR, REFramework, R.E.A.L., F.E.A.R. VR, uuvr, vorpX, geo-11 and Depth3D get stable stereo and head tracking in engines they do not own; jitter causes; the transfer plan for DayZ |
| [proprietary-engine-vr-mods.md](proprietary-engine-vr-mods.md) | Source-level survey of BladeVR, Ghost Recon Wildlands VR, BioShock Remastered VR, aliasIsolation, Star Wars Racer PCVR, UnrealVR, vrframework and Detroit VR: second eye, pose write, pose pairing, what transfers to DayZ, queued experiments |
| [tooling.md](tooling.md) | How these notes were produced: headless Ghidra project, `scripts/ghidra-decompile.sh` queries, string and import searches, pitfalls |

Conventions: `+0x..` inside a struct is a byte offset from the object start; `FUN_1400xxxxx`
names are Ghidra's; sizes are in bytes. "Verified" means observed at runtime (log, hook or
test), "decompiled" means read from Ghidra output only.
