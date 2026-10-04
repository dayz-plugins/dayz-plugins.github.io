---
name: dayz-engine-research
description: >-
  Index of the DayZ (Enfusion) engine reverse-engineering notes: which file covers rendering and the VR rendering research, events, chat and RPC, input, scripting and the Ghidra tooling, plus the conventions used in all of them.
game_build: DayZ 1.29.163709 (DayZ_x64.exe, PE timestamp 0x6A72FC58)
created: 2026-10-02T17:11+0200
last_edited: 2026-10-04T20:20+0200
---

# DayZ engine research notes

Condensed reverse-engineering findings about the DayZ (Enfusion) client, written for other
projects, modders and agents. Everything here was established on **DayZ 1.29.163709**
(`DayZ_x64.exe`, PE timestamp `0x6A72FC58`, image size `0x04406000`) unless a build is named;
addresses are RVAs relative to the image base (`DayZ+0x...`). Other builds shift every address,
so verify with the byte checks in `common/dayz_build_checks.hpp` or re-run the Ghidra queries.

| File | Topic |
| --- | --- |
| [rendering.md](rendering.md) | Frame structure, view prepare/execute/finalize, projection dispatch, camera (FrameBase), FOV, HUD scale, GUI capture, engine bugs the proxy works around, and the VR rendering research: an independent binary audit of this build's renderer, how VR mods for engines they do not own produce a second eye, jitter causes, and the ranked transfer plan for DayZ |
| [events.md](events.md) | The event system (`EventType` is a C++ RTTI pointer), how events are raised and dispatched into script and where to hook to listen or swallow, the event table with payloads, chat in and out, mod RPC, and the session natives for connect, disconnect and exit |
| [input.md](input.md) | Input system: raw input and XInput device layer, the action registry (`UAInput` records), HumanInputController action tables, focus gating, what can be written from outside |
| [scripting.md](scripting.md) | Enforce Script facts that matter for native code: native binding tables, what the client can and cannot override without a server mod, file bridge |
| [tooling.md](tooling.md) | How these notes were produced: headless Ghidra project, `scripts/ghidra-decompile.sh` queries, string and import searches, pitfalls |

Conventions: `+0x..` inside a struct is a byte offset from the object start; `FUN_1400xxxxx`
names are Ghidra's; sizes are in bytes. "Verified" means observed at runtime (log, hook or
test), "decompiled" means read from Ghidra output only, and "from the shipped scripts" means
read out of `dta/scripts.pbo`, the game's own Enforce source.
