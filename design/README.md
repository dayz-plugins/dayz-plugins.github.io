---
name: design-index
description: >-
  Index of the design documents and proposals for the plugin loader and its plugins, with the status of each.
created: 2026-10-03T08:20+0200
last_edited: 2026-10-03T08:20+0200
---

# Design documents

Proposals and decisions for the [dayz-plugins](https://github.com/dayz-plugins) loader and its
plugins. Every document carries a `status` in its front matter, so a proposal is never mistaken
for something that shipped:

| Status | Meaning |
| --- | --- |
| `proposal` | Written to be argued with. Nothing is built, and the open questions at the end are genuinely open. |
| `accepted` | Agreed, not yet built. |
| `implemented` | Built; a note at the top says where the code lives and where reality differs from the document. |
| `superseded` | Kept for the reasoning, replaced by a named document. |

| Document | Status | Topic |
| --- | --- | --- |
| [address-database.md](address-database.md) | `implemented` | Per-build game addresses as data: patterns as the source of truth, a cache keyed by executable hash, plugins asking for symbols by name. Shipped as [dayz-data](https://github.com/dayz-plugins/dayz-data). |
| [ui-overlay-and-localisation.md](ui-overlay-and-localisation.md) | `proposal` | Plugin-drawn overlays and HUD through a loader-owned `egui`, translation catalogs resolved by key, and key hints that follow the input device in use. |

Research notes that these documents rest on are in [`research/`](../research/).
