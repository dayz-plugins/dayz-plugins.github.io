# dayz-plugins.github.io

Documentation home for the [dayz-plugins](https://github.com/dayz-plugins) organisation. The
code repositories carry only a `README.md` and an `AGENTS.md`; everything else lives here.

| Folder | What is in it |
| --- | --- |
| [`research/`](research/) | Engine reverse-engineering notes: rendering, input, scripting, the Ghidra tooling, and how other projects achieve VR in engines they do not own. |
| [`design/`](design/) | Design documents and proposals for the loader and its plugins. |

## The organisation's repositories

| Repository | What it is |
| --- | --- |
| [dayz-plugin-loader](https://github.com/dayz-plugins/dayz-plugin-loader) | The loader itself (`dxgi.dll`), the plugin ABI, the SDK and the `dayz-data` reader. |
| [dayz-data](https://github.com/dayz-plugins/dayz-data) | Per-build game addresses and struct offsets: byte patterns, caches, seeds. |
| [dayz-debug-plugin](https://github.com/dayz-plugins/dayz-debug-plugin) | A plugin serving the loader's console, settings and state on a loopback socket, and `dayz-ctl` to drive it from a shell. |

Research notes state a game build at the top and were established against that build;
addresses shift between builds. Design documents carry a `status` field, so a proposal is
distinguishable from a decision that already shipped.
