---
name: dayz-engine-tooling
description: >-
  How the research notes were produced: headless Ghidra project, scripts/ghidra-decompile.sh query types, import and RTTI tricks, runtime verification.
game_build: DayZ 1.29.163709 (DayZ_x64.exe, PE timestamp 0x6A72FC58)
created: 2026-10-02T17:11+0200
last_edited: 2026-10-02T17:11+0200
---

# Tooling: how the notes were produced

## Headless Ghidra project

- Ghidra 12.1 (`/run/media/system/Data/Applications/ghidra/ghidra_12.1.4_PUBLIC`) runs inside
  the `build-box` distrobox (it has the JDK). The project lives in `build/ghidra/` and is
  created on the first run (import plus auto-analysis of `DayZ_x64.exe`, tens of minutes);
  later runs open it with `-noanalysis` in about a minute.
- `scripts/ghidra-decompile.sh` is the single entry point. Arguments are mixed freely:

| Argument | Script | Output |
| --- | --- | --- |
| `<rva>` (hex) | `DecompileRvas.java` | `build/ghidra/decomp/DayZ+0x<entry>.c` for the function containing the RVA, with its direct callers listed at the top |
| `string:<exact text>` | `FindRefs.java` | `refs-string_<text>.txt`: where the string lives, who references it; code referrers are decompiled, data referrers get their neighbouring code pointers reported (registration tables are `{name, function}` pairs) |
| `import:<symbol>` | `FindRefs.java` | references to an import symbol; ordinal-only imports appear as `Ordinal_N` |
| `addr:<rva>` | `FindRefs.java` | references to any address (an IAT slot, a global); call stubs are followed one level to the real callers |
| `table:<rva>:<count>` | `FindRefs.java` | dumps a pointer table (vtable) and decompiles every code slot |
| `vtables:<text>` | `FindRefs.java` | every RTTI class whose name contains the text, with its vftable address and slots (no decompile) |

Only one headless run can hold the project at a time; queue runs, do not start them in parallel.

## Finding things

- **Enforce natives**: the class registrators are plain functions calling
  `FUN_14031b7f0(class, ctx, "MethodName", &native, 0)` once per method. A `string:` query
  for a method name lands in the registrator and names the native directly (example: the
  `UAInput` registrator is `0x537390`, the `Input` one `0x5F76A0`).
- **Imports**: Ghidra's references to an import are usually on the IAT slot read, not on the
  external symbol. Take the IAT RVA from the PE import table (a scratch venv with `pefile`
  does this in ten lines) and query `addr:<iat rva>`. `xinput1_3.dll` is imported by ordinal.
- **RTTI survives**: MSVC vftable symbols (`Input::vftable`, `UAInputAPI::vftable`,
  `enf::RawMouseEvent::vftable`) are present, so `vtables:` enumerates whole subsystems.
- **Not vtables**: tables in `0x429xxxx` are `.pdata` unwind records (three 32-bit RVAs per
  entry), not pointer tables; `.rdata` (`0xC0xxxx`–`0xE6xxxx`) holds the real vtables.
- **Binary string searches** (`python3 -c` over the exe bytes) are faster than Ghidra for
  "does this name exist" questions: all 324 `UA*` action names and the imported DLL names came
  from that.

## Runtime verification

- The proxy logs to `dayz_openxr.log` beside `DayZ_x64.exe`; `scripts/dayz-vr-ctl.py` reads
  and sets live tunables, `watch` samples the per-frame state, `dump-eyes` writes both eye
  captures.
- `common/dayz_build_checks.hpp` carries 16-byte instruction checks per hooked address; a
  new build fails them and the hooks stay off. Add a check for every new RVA.
- `scripts/regression-run.sh` runs the headless Monado simulator rig end to end (local
  server in a container, client under Proton, scripted client/server commands).
