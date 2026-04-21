---
name: ayon-premiere
description: Adobe Premiere Pro in Ayon (`ayon-premiere`, Adobe Premiere Ayon). Integration via Adobe **CEP extension** (`io.ynput.PPRO.panel`, not `PPRS`). Auto-install via `auto_install_extension` setting (the only top-level setting in `server/settings.py`); manual install via `extension.zxp`. Panel at Window → Extensions → AYON. Two loaders: `FileLoader Premiere` (permissive — images/video/audio, smart layer) and `AECompLoader` (.aep workfiles). `PremiereWorkfileCreator` AutoCreator (product_type `workfile`). `AYON Metadata - DO NOT DELETE` bin inside `.prproj`. Extension manifest at `client/ayon_premiere/api/extension/CSXS/manifest.xml` — bump `ExtensionBundleVersion` per addon update. No `PremiereHost` class — integration is CEP panel + WebSocket bridge (`WEBSOCKET_URL`) via `PremiereAddon(AYONAddon, IHostAddon)`. Use when integrating Premiere, debugging extension install, advising artists, or bumping the addon.
---

# Adobe Premiere Pro in Ayon

Repo: `github.com/ynput/ayon-premiere`. Integrates via an Adobe
**CEP extension** (JavaScript panel) driven by the Python client side
via a WebSocket bridge. Source-confirmed against develop branch.

## Access

### Automatic install

Enable the addon setting `auto_install_extension` (default `True`).
The `InstallAyonExtensionToPremiere` pre-launch hook deploys the CEP
extension into AppData on Premiere launch and keeps its version
aligned with the addon's `ExtensionBundleVersion`.

Install path (Windows):
```
C:\Users\<user>\AppData\Roaming\Adobe\CEP\extensions\io.ynput.PPRO.panel
```
(macOS: `~/Library/Application Support/Adobe/CEP/...`)

Panel Extension Bundle ID: **`io.ynput.PPRO.panel`** (not `PPRS`).

### Manual install

Extension packaged at `client/ayon_premiere/api/extension.zxp`. Install
via Adobe's `ExManCmd`.

### Open the panel

**Window → Extensions → AYON**.

## Integration shape

**No `PremiereHost` class.** Integration is CEP panel (JS+JSX) + Python
launcher-side addon:

- `PremiereAddon(AYONAddon, IHostAddon)` in `client/ayon_premiere/addon.py`
- Implements `add_implementation_envs()` (sets `WEBSOCKET_URL`,
  `AYON_LOG_NO_COLORS`), `get_workfile_extensions()` → `[".prproj"]`,
  `get_launch_hook_paths()`
- Panel calls back into the launcher over WebSocket for Creator/Publisher/Loader UIs

## Tools in the panel

Workfiles · Creator/Publisher · Loader · Scene Inventory.

## Auto-published workfile

`PremiereWorkfileCreator` (`plugins/create/workfile_creator.py`) is an
**AutoCreator** with `product_type = "workfile"`. It registers the
workfile instance automatically on every reset.

**Not** `workfileCompositing` — older KB versions had this wrong.

## Loaders (source-confirmed)

| Loader class | File | Scope |
|--------------|------|-------|
| `FileLoader` | `plugins/load/load_file.py` | Images/movies/audio. `product_base_types = {"*"}`; extension filter via `IMAGE_EXTENSIONS + VIDEO_EXTENSIONS` (+ audio exts) |
| `AECompLoader` | `plugins/load/load_aftereffects_comp.py` | `.aep` workfiles. `product_base_types = {"workfile"}` |

The artist help article lists "image / plate / render / prerender /
review / audio" — that's a human simplification of what `FileLoader`
permissively accepts.

**Loaded images must stay as smart layers** to remain updatable
(rasterising breaks versioning, enforced by convention not code).

## Metadata storage

`AYON Metadata - DO NOT DELETE` bin inside the `.prproj` — referenced
in loader docstrings. Don't rename or delete.

## Addon layout (source-confirmed)

```
ayon-premiere/client/ayon_premiere/
├─ __init__.py                   # exports PremiereAddon
├─ addon.py                      # PremiereAddon(AYONAddon, IHostAddon)
├─ api/
│  ├─ extension.zxp              # packaged CEP panel
│  └─ extension/
│     ├─ index.html              # panel HTML
│     ├─ js/main.js              # panel JS entry
│     ├─ jsx/hostscript.jsx      # ExtendScript bridge
│     └─ CSXS/manifest.xml       # io.ynput.PPRO.panel / EBV=1.1.10
├─ plugins/
│  ├─ create/workfile_creator.py # PremiereWorkfileCreator
│  ├─ load/load_file.py          # FileLoader
│  ├─ load/load_aftereffects_comp.py  # AECompLoader
│  └─ publish/                   # CollectCurrentFile + workfile collectors
└─ hooks/
   ├─ pre_launch_install_ayon_extension.py  # InstallAyonExtensionToPremiere
   └─ pre_launch_args.py
```

**No**: `plugins/inventory/`, `startup/`, `PremiereHost` class.

## Settings (source-confirmed)

Exactly **one** top-level field in `server/settings.py`:

| Field | Type | Default | Title |
|-------|------|---------|-------|
| `auto_install_extension` | `bool` | `True` | "Install AYON Extension" |

Accessed in code as `project_settings["premiere"]["auto_install_extension"]`.

## Caveats

- **`io.ynput.PPRO.panel`** — not `PPRS.panel`.
- Bump `ExtensionBundleVersion` in `manifest.xml` per release.
- **No `PremiereHost`** — don't look for one.
- CEP vs UXP: Premiere still CEP. UXP integrations for newer Adobe
  apps would need a separate addon.
- Smart-layer requirement: convention only, not code-enforced.

## Deeper reading

- `23-premiere.md` — full file with Sources
- Skill `ayon-addon-development` — `AutoCreator` pattern

## Sources

- <https://help.ayon.app/en/articles/3607676-working-with-premiere-in-ayon>
- <https://help.ayon.app/en/articles/1663762-premiere-addon-settings>
- <https://github.com/ynput/ayon-premiere/blob/develop/server/settings.py>
- <https://github.com/ynput/ayon-premiere/blob/develop/client/ayon_premiere/addon.py>
- <https://github.com/ynput/ayon-premiere/blob/develop/client/ayon_premiere/api/extension/CSXS/manifest.xml>
- <https://github.com/ynput/ayon-premiere/blob/develop/client/ayon_premiere/plugins/create/workfile_creator.py>
- <https://github.com/ynput/ayon-premiere/blob/develop/client/ayon_premiere/plugins/load/load_file.py>
- <https://github.com/ynput/ayon-premiere/blob/develop/client/ayon_premiere/hooks/pre_launch_install_ayon_extension.py>
