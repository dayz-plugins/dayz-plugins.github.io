---
name: dayz-engine-scripting
description: >-
  Enforce Script facts that matter for native code and client-only mods: native binding tables, client versus server authority, spawning flags, language pitfalls and the file bridge.
game_build: DayZ 1.29.163709 (DayZ_x64.exe, PE timestamp 0x6A72FC58)
created: 2026-10-02T17:11+0200
last_edited: 2026-10-02T17:11+0200
---

# Enforce Script facts for native code

What a client-side mod (`-mod=@...`, `scripts/` PBO with `3_Game`..`5_Mission` modules) can
and cannot do, and how it meets native code. Scripts are in `dta/scripts.pbo`; the file is
greppable as raw bytes.

## Natives and registration

- Every `proto native` method is bound by a registrator function that calls
  `FUN_14031b7f0(class, ctx, "Name", &function, 0)` per method; the method name string
  therefore leads straight to the native implementation (see tooling.md).
- `UAInput` script objects are the engine's action records themselves (no wrapper), see
  input.md. `Input` (`GetGame().GetInput()`) wraps the `Input` system object at `+0x28`.

## Client authority

- `HumanInputController.Override*` (`OverrideMovementSpeed/Angle`, `OverrideAimChangeX/Y`,
  `OverrideRaise`, `OverrideMeleeEvade`, `OverrideFreeLook`, `Override3rdIsRightShoulder`;
  enum `HumanInputControllerOverrideType { DISABLED, ENABLED, ONE_FRAME }`) apply on the
  caller only. In multiplayer 1.29 the owner's movement overrides alone do not move the
  player on the server (dayz-mcp PR 191 measured 0 m): the server must apply the same
  overrides per consumed move, so movement through overrides needs a server mod.
  `OverrideRaise` is honoured client-side (verified in this project).
- Vehicle control natives `Car.SetSteering/SetThrottle/SetBrake` called from
  `CarScript.OnUpdate` on the driving client beat the engine's own driver input (verified).
- Starting vehicle commands (`StartCommand_Vehicle`, `GetOutVehicle`) only works from the
  player's `CommandHandler` tick.
- Server-authoritative user actions (lights, horn, interactions) go through
  `ActionManagerClient.PerformActionStart(action, target, item)` after `action.Can(...)`;
  engine start/stop for physics cars is client-side (`EngineStart/EngineStop`).
- Spawning: `CreateObjectEx(name, pos, flags)` needs `ECE_SETUP (2)` for the full entity setup
  (the vanilla ObjectSpawner uses `ECE_SETUP | ECE_UPDATEPATHGRAPH (32) | ECE_CREATEPHYSICS
  (1024)`); without it a car has no collision. `ECE_PLACE_ON_SURFACE = 128`.
- Class existence checks must cover `CfgVehicles`, `CfgWeapons`, `CfgMagazines`, `CfgAmmo`.

## Language pitfalls

- A `return a || b || c;` spread over several lines fails to compile ("Missing ;"); write
  separate statements.
- No sockets or FFI: native code and scripts exchange data through files under
  `$profile:dayzvr/` (this project's bridge: `vr.txt` native→script, `game.txt` script→native,
  `client_cmd.txt` commands), polled at 10 Hz from `MissionGameplay.OnUpdate`.
- `Print` goes to `script.log` in the profile directory; server script errors appear as
  `SCRIPT (E)` lines and stop the whole mission module from loading.
