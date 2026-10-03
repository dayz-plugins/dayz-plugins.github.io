---
name: dayz-engine-scripting
description: >-
  Enforce Script facts that matter for native code and client-only mods: the ScriptModule class and its native bindings, how the engine finds script modules, client versus server authority, spawning flags, language pitfalls and the file bridge.
game_build: DayZ 1.29.163709 (DayZ_x64.exe, PE timestamp 0x6A72FC58)
created: 2026-10-02T17:11+0200
last_edited: 2026-10-03T09:40+0200
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
- Two registrators, side by side in every class's registration: `0x31B7F0` for instance
  methods and `0x31B790` for static ones, the latter with an extra byte flag at `rsp+0x28`.
  Both take `(class, ctx, name, function, flags)`. They are in the address database as
  `script.native_register_method` and `script.native_register_static`.
- The registration function returns early through `0x31B910` with the class name in `rcx`
  when the class cannot be created, so a failed registration is one log line, not a crash.

## The ScriptModule class

The script-visible gateway to compiling and calling Enforce code. Its registration is at
`0x2D85FB` (`script.scriptmodule_registrator`), and it binds, in order:

| Script method | Native | Note |
| --- | --- | --- |
| `Call` | `0x31AAA0` | |
| `CallFunction` | `0x31ACE0` | |
| `CallFunctionParams` | `0x31AEF0` | |
| `Release` | `0x2E7420` | |
| `LoadScript` | `0x2D9A30` | static; **a stub on retail**, see below |

The same function registers the script global `g_Script`, bound to the engine's own
`ScriptModule` instance held at `0xFF1B20` (`script.script_module_instance`). RTTI confirms
the C++ names: `.?AVScriptModule@enf@@`, `.?AVScriptModulePathClass@enf@@`, plus
`WeakPtrBase<ScriptModule>` and `WeakPtrTracker<ScriptModule>`.

### `ScriptModule.LoadScript` is not gated, it is gutted

The whole native body on a retail client is six instructions:

```text
1402d9a30  sub    rsp, 0x28
1402d9a34  mov    rcx, [0x140ff1b30]                  ; logger
1402d9a3b  lea    rdx, ["'ScriptModule.LoadScript' can't be called on Retail Client!"]
1402d9a42  mov    r8d, 1                              ; severity
1402d9a48  call   0x1403186a0                         ; log
1402d9a4d  xor    eax, eax                            ; return null
```

There is no branch and no flag to flip: the retail build ships a different function for this
binding, not the real one behind a check. So **patching a boolean somewhere cannot turn script
loading on**, and the two ways left are to replace what the registrator bound (rebind the name
or detour `0x2D9A30`) or to go in through the config path below, which is the one the engine
itself still uses.

## How the engine finds script modules

- `0x4CA5C0` (`script.module_config_read`) reads five keys from a mod's `CfgMods` entry in
  this order: `engineScriptModule`, `gameLibScriptModule`, `gameScriptModule`,
  `worldScriptModule`, `missionScriptModule` (string reads at `0x4CA5FE`, `0x4CA666`,
  `0x4CA6C9`, `0x4CA728`, `0x4CA787`). Each one is handed, with the key name in `r9`, to the
  common helper `0x4CA7E0` (`script.module_config_add`).
- `CfgMods` itself is referenced five times: `0x43F6F2`, `0x43F75F`, `0x43F9B1`, `0x43FA0B`
  (the mod setup function) and `0x5977EB`.
- `ScriptModulePathClass` is constructed at `0x146444` and has a second site at `0x147440`;
  the `ScriptModules` config class name is read at `0x1454C7`.
- Taken together: the machinery that compiles a directory or PBO of `.c` files into a module
  is intact and is driven entirely by config. Nothing in it needs the script-facing
  `LoadScript`.

### What a `<game_dir>/scripts/` loader still needs

For the planned `dayz-enforcer-plugin`, the open questions are narrower than they were:

1. The signature and calling convention of `0x4CA7E0` — what it does with the key name and
   where it puts the resulting module path. That decides whether a plugin can simply call it
   with a synthesised path, which would be the smallest possible intervention.
2. Whether module registration is still accepted after startup, or whether `0x4CA5C0` runs
   once before the mission module is built. If it is startup-only, the hook has to happen
   from `DllMain`/first DXGI call rather than from a hotkey.
3. The real compile entry point reached from `0x4CA7E0`, which is what a rebound `LoadScript`
   would have to call to be useful at runtime.

None of this is settled yet; what is settled is that the retail stub is a dead end and the
config path is not.

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
