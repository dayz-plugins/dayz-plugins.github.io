---
name: ui-overlay-and-localisation
description: >-
  Design for plugin-drawn UI and HUD overlays, a loader-owned translation catalog, and key hints that follow the input device in use. One pipeline: the loader owns egui, plugins talk to it through an immediate-mode C ABI, text comes from catalogs by key, and a hint is an action resolved to a binding, a device and a glyph.
status: proposal
created: 2026-10-03T08:15+0200
last_edited: 2026-10-03T08:15+0200
---

# UI overlays, localisation and input-aware key hints

## The problem

Three requests that arrived together, and that turn out to be one feature:

1. **Plugins need to show things.** A HUD element, a settings panel, a VR calibration dialog.
   Today the loader draws nothing at all: `--console` allocates a Windows console window, so
   the "in-game console" is an alt-tab away and the key under Escape does nothing.
2. **Mods need translated strings**, without each one inventing a file format, a language
   setting and a fallback rule.
3. **Key hints must follow the device in use** — "Left Mouse Click" with a mouse, "Right
   Trigger" once a controller is in use — which means hints cannot be literal strings in a
   plugin at all.

They are one pipeline because a hint is a *translated label plus a glyph, drawn by the
overlay, chosen by the current device*. Build them separately and the seams end up in the
plugin API, where they are expensive to move.

## Recommendation in one paragraph

**The loader owns one UI library and plugins never see it.** Take `egui`: it is pure Rust,
cross-compiles today, and emits surface-independent meshes rather than drawing, which is what
makes the same plugin UI work in a flat overlay now and on a world-space quad in VR later.
Plugins reach it through an immediate-mode facade over the C ABI — `on_ui(frame)` plus
`ui_button`, `ui_slider` and friends, keyed by an opaque frame token — so the SDK can make
plugin code read like Dear ImGui while the loader keeps the device, the frame lifetime and the
fault guard. Text crosses that ABI as *catalog keys*, never as sentences, so translation is a
file the loader resolves and not a concern in the plugin. A key hint is requested by **action
name**: the loader resolves action to current binding, binding to the device actually in use
(the engine already tracks it), and device to a glyph plus a translated label. Build in the
order overlay, catalogs, hints; the overlay's first customer is the loader's own console,
which means the machinery is exercised before any plugin depends on it.

## Build order, and why it is not negotiable

| Order | Why it has to be here |
| --- | --- |
| 1. Overlay renderer and input | Nothing else is visible without it. Its first customer is the in-game console, so it is proven by the loader before a plugin API is frozen. |
| 2. Localisation catalogs | Needed before any UI text is written, because retrofitting keys into a plugin's strings means touching every plugin. Also decides the font and shaping requirements. |
| 3. Key hints | Needs a binding registry (exists), a device source (research done, see below), a glyph atlas (part 1) and labels (part 2). Last for a reason. |

## Part 1 — the overlay

### Library choice, measured

Both candidates were built for the real target inside the `build-box` container, because a
C++ dependency that cannot cross-compile has already cost this project a day (`retour` and its
`libudis86-sys`).

| Candidate | `cargo xwin build --target x86_64-pc-windows-msvc` |
| --- | --- |
| `egui` 0.35 + `egui-directx11` 0.13 + `windows` 0.62 | builds clean, 30 s cold, 5.1 MB cdylib, no C or C++ in the dependency tree |
| `imgui` 0.12 (cimgui, C++ through `cc`) | also builds clean, 24 s |

So Dear ImGui is **not** disqualified on build grounds, which was the initial assumption and
was wrong. It loses on three others:

- **No maintained Direct3D 11 renderer.** `imgui-dx11-renderer` is stuck at `imgui` 0.8; the
  actively maintained binding (`dear-imgui-rs` 0.18, Dear ImGui 1.92) ships SDL3, WGPU and
  Bevy backends and nothing for a device obtained by hooking someone else's swapchain.
- **C++ FFI per widget**, against the grain of the workspace's `unsafe_code = "deny"`.
- **No complex-script shaping.** Font atlases are configured by glyph range per language. egui
  0.35 pulls in `harfrust`, a pure-Rust HarfBuzz port, so Cyrillic, CJK and Arabic are a font
  question rather than a rewrite. Since part 2 of this document exists to produce translated
  text, that is decisive.

A hand-written immediate-mode UI was considered and rejected: the work in a UI library is
text layout, shaping, editing and focus, not rectangles.

### Why it carries over to VR unchanged

egui does not render. It emits `[ClippedPrimitive]` meshes plus texture deltas, and the
caller rasterises them. That single property is the reason this design survives VR:

- **Desktop**: rasterise to the backbuffer inside the existing `Present` hook.
- **VR**: rasterise the same meshes once into an offscreen render target and submit it as a
  quad layer to the compositor; synthesise pointer events from the controller ray into
  `egui::RawInput`.

Only the render target and the input source differ. No plugin changes, and no plugin *can*
change, because none of them holds a device. That is the whole argument for the loader owning
the library.

### The plugin-facing ABI

egui's types cannot cross a C ABI, and sharing an `egui::Context` between separately built
DLLs is not safe: Rust has no stable ABI, so a shared context would mean every plugin rebuilds
whenever the loader bumps egui. That is strictly worse than the existing `API_VERSION` rule,
which only forces a rebuild when the ABI itself changes.

Instead, an immediate-mode facade. The loader calls the plugin once per frame with an opaque
frame token; the plugin calls back through host functions that take it:

```rust
// What a plugin writes, through the SDK wrapper.
fn on_ui(&self, ui: &Ui) {
    ui.window("#vr.panel.title", |ui| {
        if ui.button("#vr.recenter") {
            self.recenter();
        }
        ui.slider("#vr.ipd", "vr.ipd", 0.05..=0.08);   // bound to the setting by name
        ui.label_with_hints("#vr.hint.recenter");       // "Press {key:vr.recenter} to recenter"
    });
}
```

```c
/* What crosses the ABI. */
typedef struct UiFrame UiFrame;                 /* opaque, valid only during on_ui */
Status ui_window_begin(void* host, PluginHandle p, UiFrame* f, Str title_key, UiRect anchor);
Status ui_window_end  (void* host, PluginHandle p, UiFrame* f);
Status ui_label       (void* host, PluginHandle p, UiFrame* f, Str text_key, const UiArg* args, size_t n);
Status ui_button      (void* host, PluginHandle p, UiFrame* f, Str text_key, bool* out_clicked);
Status ui_slider_setting(void* host, PluginHandle p, UiFrame* f, Str text_key, Str setting);
```

Properties worth stating, because they are the point:

- **One FFI call per widget.** At a few hundred widgets a frame this is noise next to the
  rasterisation.
- **The loader opens and closes the frame**, so a plugin that panics mid-UI cannot leave the
  context half-built. The existing per-callback panic and fault guard applies unchanged, and a
  plugin that faults in `on_ui` is disabled like any other.
- **Widgets bind to settings by name** (`ui_slider_setting`), so a settings panel needs no
  state in the plugin and validation stays in one place.
- **Text is a key, not a sentence.** See part 2. A plugin that passes a literal string still
  works — a key that is not in any catalog renders as itself — which keeps the quick case
  quick and the correct case barely longer.

A second, lower tier stays available for a plugin that genuinely wants to paint: submit a flat
list of draw commands (rect, rounded rect, nine-slice, line, text run, image) in layout space.
The loader rasterises those through the same path, which also means they appear on the VR quad
for free.

### Input plumbing, the one missing piece

The loader currently polls `GetAsyncKeyState` (`win/input.rs`), which is sufficient for
hotkeys and insufficient for a UI: no text input, no mouse wheel, no event ordering. A UI
needs a WndProc subclass on the output window the loader already receives in `on_swapchain`,
translating messages into `egui::RawInput`. Standard overlay technique, and the place where
focus policy lives:

| Panel focus mode | Behaviour |
| --- | --- |
| `passthrough` | Drawn, never takes input. HUD elements. |
| `capture` | Takes mouse and keyboard; game input is suppressed while it is open. |

Suppression has a known route: the loader already knows whether the game window has focus
(`input.has_game_focus`, `input.focus_counter`), and the UAInput getter hooks documented in
[research/input.md](../research/input.md) are how a captured frame stops reaching gameplay.
Non-global hotkeys (which already exist) toggle panels.

### First customer: the loader's own console

Before the plugin ABI is frozen, the overlay draws the console that today lives in a separate
Windows window: same command set, same output buffer, bound to a hotkey. It exercises text
input, scrolling, focus capture and theming, and it answers the question "how do I open the
console in-game" with a key instead of an alt-tab.

## Part 2 — localisation

### Shape

- **A plugin ships its own defaults compiled in** and registers them during `start`
  (`locale_register(bytes, "en")`), so a plugin with no files on disk is fully functional and
  English.
- **The loader overlays files** from `plugin-loader/locale/<plugin>.<lang>.*`, which is what a
  translator edits. A `locale reload` console command re-reads them, turning translation into
  a one-second loop instead of a restart.
- **Language selection**: `loader.language`, default `auto` → the game's own `-language=` or
  config, then the operating system, then `en`. `--language=de` overrides for one run.
- **Format**: Fluent (the `fluent` crate), loader-side only, for real plural and gender rules.
  It never appears in the ABI, so the decision is reversible; an ICU-lite subset
  (`{count}` with `[one]`/`[other]`) is the fallback if the dependency is unwelcome.

### The convention that makes it nearly free

Adopt Enforce Script's `#` prefix: any `SettingDesc.title`, hotkey title, command help or UI
string beginning with `#` is a catalog key the loader resolves. Then every settings panel,
every `help` listing and every console line is translated with **no new plugin API**, in a
syntax DayZ modders already recognise from `#STR_` keys.

```c
Status text_get(void* host, PluginHandle p, Str key, const UiArg* args, size_t n,
                uint8_t* buf, size_t cap, size_t* out_len);   /* same shape as setting_get */
Status locale_register(void* host, PluginHandle p, Str lang, Bytes catalog);
```

### Fonts are part of this, not an afterthought

A translation the atlas has no glyphs for is a row of boxes. The loader owns the font stack:
a default face, a fallback chain per script, and shaping through `harfrust`. This is the
reason the library choice in part 1 is settled by the existence of part 2.

### Later, needs research

The game's own `stringtable` lookup, so a plugin can reuse vanilla terms ("Ammo", "Bandage",
key names) rather than retranslating them. We know enough about the script side to look; the
lookup function is not mapped yet.

## Part 3 — key hints

### Ask by action, never by key

```rust
let hint = host.key_hint("vr.recenter")?;   // KeyHint { label, glyph }
```

The loader resolves, in order: **action → current binding** (the hotkey registry, including
the user's override) **→ device in use → style profile → glyph + translated label**. A plugin
never names a key, which also means a rebinding updates every hint in every plugin at once.

### The device source is already mapped

From [research/input.md](../research/input.md), on build 1.29.163709:

| Fact | Where |
| --- | --- |
| Last input device type (`EInputDeviceType`, 1 = mouse and keyboard) | `Input+0xE0`, and the engine queues a `LastInputDeviceChangeEvent` on change |
| Gamepad connected flags, active pad | `Input+0x8195..`, `Input+0x81C8` |
| `Input.IsEnabledGamepad()` | `0x5F6CB0` |

So `host.input_device()` and an `on_input_device_changed` callback read what the engine already
decided, rather than inferring it from which key last arrived. One cheap read per frame.

### Controller style

XInput cannot report a brand. Use the HID `VID/PID` path (already in the research notes) with
`loader.controller_style = auto|xbox|playstation|switch|deck` as the escape hatch, because
third-party pads lie and the user knows what is in their hands.

### Inline markup, so hints work in prose

A hint alone is rarely what you want. Let any UI string carry them:

```
vr.hint.recenter = Press {key:vr.recenter} to recenter your view
```

The loader substitutes a glyph during layout and a label in plain-text contexts such as the
console, so one string serves both. This is the point where parts 1, 2 and 3 meet: the markup
lives in the catalog, the glyph comes from the overlay's atlas, and the binding comes from the
device.

### Scope limit, stated now

This covers **the loader's own actions**. Hinting a *vanilla* action ("whatever is bound to
`UAFire`") needs the engine's current binding table read back; the UAInput research names the
actions but not the live bindings. That is a research task, not a part of this proposal.

## ABI additions in one place

All of this is additive, so it is one version bump (`API_VERSION` 3 → 4) with no field
changing meaning:

| Addition | Kind |
| --- | --- |
| `ui_*` immediate-mode functions, `ui_draw_list` | host functions |
| `on_ui(frame)` | plugin callback |
| `text_get`, `locale_register` | host functions |
| `key_hint`, `input_device` | host functions |
| `on_input_device_changed` | plugin callback |
| `#`-prefixed titles resolved through the catalog | convention, no new function |

## What this costs

- The loader DLL grows by roughly 4–5 MB (egui plus default fonts), measured from the probe
  build. It is already 1.4 MB; this is not a constraint anyone will feel.
- One WndProc subclass, which is new attack surface for bugs in the "game stops receiving
  input" category. Mitigated by the focus modes above and by the console being the first
  consumer, where the failure is obvious rather than subtle.
- `egui-directx11` 0.13 pins egui 0.35 while 0.36 is current, and it is a small
  single-maintainer crate. Accept it to get moving; the renderer is a few hundred lines
  against a device we already own, so vendoring it is a contained fallback rather than a
  rewrite.

## Open questions

- **Per-eye UI placement in VR.** A quad layer is right for panels; a HUD element may want to
  be head-locked, world-locked or weapon-locked. Does the anchor vocabulary need a third axis
  for that, or does the VR plugin own the transform and the UI plugin stay flat?
- **Who owns the glyph atlas for controller buttons?** Shipping it in the loader is simplest
  and keeps hints consistent; letting a plugin add glyphs invites eight different Xbox "A"
  buttons.
- **Fluent or the ICU-lite subset.** Fluent is correct and is another dependency with its own
  parser; the subset covers most HUD text. Decide before the first catalog is written, because
  the files are what is expensive to migrate, not the code.
- **Does `on_ui` run on the render thread?** It does if it is called from the `Present` hook,
  which means a plugin's UI code shares that thread's constraints. The alternative is building
  the frame on a loader thread and only rasterising in `Present`, at the cost of one frame of
  latency and a copy.
- **Text input while the game has focus.** A captured panel must receive characters without
  the game also acting on them. The UAInput hooks suppress gameplay actions; whether anything
  else in the engine consumes `WM_CHAR` first is unverified.
