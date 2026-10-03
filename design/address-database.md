---
name: address-database
description: >-
  Design for the per-game-version address and offset database: patterns as the source of truth, resolved addresses cached per executable hash, the loader resolving symbols by name so a game update touches the loader's data and not every plugin.
status: implemented
created: 2026-10-03T07:00+0200
last_edited: 2026-10-03T07:30+0200
---

> [!NOTE]
> Implemented on 2026-10-03 as [dayz-data](https://github.com/dayz-plugins/dayz-data) (the
> data) plus `crates/dayz-data` and `tools/dayz-data-tool` in the loader (the code). The name
> is `dayz-data` rather than `dayz-offsets`, and the game directory holds it at
> `dayz-plugins/data/`. Everything below describes what was built, except where an open
> question says otherwise.

# Address and offset database

## The problem

Native plugins reach into the game at fixed addresses. The predecessor project kept those in
a C++ header (`common/dayz_build_profiles.hpp`) guarded by 16-byte instruction checks
(`common/dayz_build_checks.hpp`), compiled into the binary. Every DayZ update invalidated it,
and because each plugin carried its own copy, a game update meant rebuilding and re-releasing
everything.

The goal: **a game update costs a data file, not a release of every plugin.**

## Recommendation in one paragraph

Store *patterns*, not addresses, as the source of truth. Keep resolved addresses as a cache
keyed by the SHA-256 of `DayZ_x64.exe`. The loader resolves every symbol once at startup,
from the cache when the hash is known and by scanning the patterns when it is not, verifies
each hit against a short expected-byte check, and exposes the results to plugins **by name**.
A plugin declares the symbols it needs and never sees an address literal. A game update then
has three possible outcomes, in descending order of likelihood: the patterns still match and
nothing needs doing beyond the loader writing a new cache entry; one pattern broke and a data
file fixes it; the function genuinely changed and research is needed. Only the third case is
real work, and in none of them does a plugin get rebuilt.

Address-only databases, which is what MelonLoader-era offset dumps for Unity games tended to
be, skip the first two outcomes and make every update a manual data update. Pattern scanning
costs a few milliseconds of startup and removes most of that work.

## Where it lives

A separate data repository, `dayz-data`, holding JSON only and no code.

Reasons for the split: data can be corrected without releasing the loader, a contributor can
open a pull request against it without a Rust toolchain, and its validation is a schema check
rather than a build. The loader ships a bundled copy of the data it was released with, and
also reads a user-side directory that wins over the bundle, so a user can drop in a fix for a
brand-new build the day it ships.

```
dayz-data/
├── patterns.json              version independent signatures, the source of truth
├── builds/
│   └── 1.29.163709.json       resolved cache for one executable hash
├── seeds/
│   └── 1.29.163709.json       the addresses a person established, input to the generator
├── schema/
│   ├── patterns.schema.json
│   └── build.schema.json
├── README.md
└── AGENTS.md
```

In the game directory the loader looks in `dayz-plugins/data/`, which `build.sh --deploy`
fills from a checkout of the data repository. A cache entry the loader generated itself is
written there, never back into the repository.

## Keying

**SHA-256 of the executable**, in full, as the primary key. The game ships one executable, the
hash is exact, and hashing 70 MB costs tens of milliseconds once per launch. The file name is
the human-readable build number; the hash lives inside it, so two builds that share a version
string cannot collide.

Secondary metadata is for humans and for sanity checks, not for matching: PE timestamp, image
size, file size, and the date the entry was verified. A build whose hash is unknown but whose
PE timestamp matches a known entry is still treated as unknown; it only earns a log line
suggesting which entry to start from.

## File shapes

`builds/<version>.json`, the resolved cache:

```json
{
  "schema": 1,
  "build": {
    "version": "1.29.163709",
    "executable": "DayZ_x64.exe",
    "sha256": "0000000000000000000000000000000000000000000000000000000000000000",
    "file_size": 71041024,
    "pe_timestamp": "0x6A72FC58",
    "image_size": "0x04406000",
    "verified": "2026-10-03",
    "source": "ghidra"
  },
  "symbols": {
    "render.frame": {
      "rva": "0x8E77C0",
      "check": "48 8B C4 48 89 58 08 48 89 70 10",
      "note": "Per-frame work; builds visibility and draw lists."
    },
    "camera.manager": { "rva": "0x1007CE0", "kind": "global" }
  },
  "offsets": {
    "framebase.rotation": { "value": "0x08", "note": "3x3 basis, honoured by the renderer." },
    "framebase.translation": { "value": "0x2C", "note": "Ignored by the renderer." }
  }
}
```

`patterns.json`, the source of truth:

```json
{
  "schema": 1,
  "symbols": {
    "render.frame": {
      "patterns": ["48 8B C4 ?? 48 89 58 08 48 89 70 10"],
      "note": "First match in .text; the prologue is stable across 1.2x builds.",
      "since": "1.26"
    },
    "camera.manager": {
      "patterns": ["48 8B 0D ?? ?? ?? ?? 48 85 C9 74 ??"],
      "resolve": { "kind": "rip_relative", "offset": 3 },
      "note": "Reached through a RIP-relative load rather than matched directly."
    }
  }
}
```

Three field groups carry the design:

- **`patterns`** is a list, tried in order. A second entry is how a signature that changed
  shape in a newer build keeps working for older ones.
- **`resolve`** says what to do with a hit: use it directly, follow a RIP-relative operand to
  a global, or step back to a function start. Globals and call targets are the common cases
  and both need this.
- **`check`** in a cached entry is the predecessor project's instruction check, kept: a cache
  entry whose bytes no longer match is discarded rather than used, which turns a stale file
  into a clean "unresolved" instead of a crash at a wrong address.

Names are `subsystem.thing`, matching how the research notes are organised: `render.*`,
`camera.*`, `input.*`, `gui.*`, `script.*`. Struct field offsets are separate from code
addresses because they are a different kind of thing and resolve differently.

## What the loader does with it

1. Hash the executable.
2. Load the matching build file; on a miss, scan `patterns.json` against the mapped image.
3. Verify every hit against its `check`, where one exists.
4. Build a name-to-address table, and log one line per symbol with its source: cached,
   scanned, or unresolved.
5. On a scan that resolved symbols for an unknown hash, write a candidate build file into
   `dayz-plugins/offsets/` marked `"source": "scan"` and unverified, so it can be reviewed and
   contributed upstream.

Plugins get two things in the host API: a lookup by name, and a declaration of requirements.
The declaration is the part that matters for robustness. A plugin states at start which
symbols it cannot work without; the loader refuses to start it with a log line naming the
missing symbol, instead of letting it run and dereference a null. A plugin that can degrade
asks for the symbol and handles absence itself.

This is also where the version-independence claim gets its teeth: a plugin that only ever
names symbols has no build-specific code in it at all, so it keeps working across game
updates without being touched.

## What was built, measured

Against the real 1.29.163709 executable, all 21 seeded symbols and 11 offsets resolve from
the cache with zero issues, and all 17 code symbols also resolve by pattern scanning alone,
at the same addresses, with the cache removed. That second result is the one that matters: it
is the mechanism a game update depends on, exercised rather than assumed.

Two things turned out differently from the sketch above. A global gets no byte check and no
pattern, because its bytes in the file are initialisation data rather than what memory holds
at runtime, so a check over one would fail on every launch. And generated patterns are
literal byte runs, since deriving wildcards needs a length-disassembler the tool does not
have; they are correct for the build they came from and need operand bytes wildcarded by hand
to survive an update.

## Open questions

- **Scan cost.** Not yet measured in the game. The image is 68 MB mapped and scanning only
  covers symbols the cache does not, so a known build scans nothing; the cost only appears
  after an update. Mitigation if it bites: scan lazily on first lookup.
- **Who owns a symbol's name?** A plugin that needs something the database does not have
  should be able to carry its own pattern file rather than wait for an upstream entry. This
  argues for a per-plugin `offsets/<plugin>.json` merged into the same table under a
  namespace, which keeps experiments out of the shared data.
- **Signing or trust.** The data directory is read from the game folder, and a pattern file
  can point the loader at an arbitrary address. That is no worse than a plugin DLL, which can
  do anything already, so probably nothing is needed; worth stating rather than assuming.
- **Generation.** The Ghidra headless scripts in the loader repository already produce
  decompiled output and reference lists. Turning a verified address into a pattern is
  mechanical and should become a small tool rather than a manual step.
