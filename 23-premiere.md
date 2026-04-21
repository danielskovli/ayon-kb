# 23 — Adobe Premiere Pro in Ayon

Repo: `github.com/ynput/ayon-premiere`. Premiere integrates via an Adobe
**CEP extension** (JavaScript panel inside Premiere) driven from the
Python client side. Source-confirmed 2026-04-21 against develop branch.

## Extension install & access

### Automatic install

Enable the addon setting `auto_install_extension` (default `True`).
The addon's `InstallAyonExtensionToPremiere` pre-launch hook deploys
the CEP extension into the user's AppData when Premiere launches, and
keeps the installed version aligned with the addon version by
comparing `ExtensionBundleVersion` in `manifest.xml`.

Install path (Windows):
```
C:\Users\<user>\AppData\Roaming\Adobe\CEP\extensions\io.ynput.PPRO.panel
```
(macOS has the `~/Library/Application Support/Adobe/CEP/...` equivalent.)

The panel's Extension Bundle ID is **`io.ynput.PPRO.panel`** (note
`PPRO`, not `PPRS`) — declared in `client/ayon_premiere/api/extension/CSXS/manifest.xml`.

### Manual install

The addon ships the packaged extension at:
```
client/ayon_premiere/api/extension.zxp
```
(Launcher storage path: `%LOCALAPPDATA%\Ynput\AYON\addons\premiere_X.X.X\ayon_premiere\api\`)

Install via Adobe's `ExManCmd` or a CEP-enabled installer.

### Access in Premiere

**Window → Extensions → AYON** opens the docked panel.

## Integration shape

There is **no `PremiereHost` class**. Integration is driven entirely by:

- The CEP extension (JS + JSX `ExtendScript` bridge) running inside Premiere
- `PremiereAddon(AYONAddon, IHostAddon)` on the launcher side
  (`client/ayon_premiere/addon.py`)

`PremiereAddon` implements:

- `add_implementation_envs()` — sets `AYON_LOG_NO_COLORS`,
  `WEBSOCKET_URL`
- `get_workfile_extensions()` → `[".prproj"]`
- `get_launch_hook_paths()` → `client/ayon_premiere/hooks/`

The CEP panel uses a WebSocket bridge (the `WEBSOCKET_URL` env) back
to the launcher process for Creator/Publisher/Loader UIs.

## Tools in the panel

The panel mirrors the standard Ayon tool set:

- **Workfiles** — manages saving `.prproj` files per Ayon's work template.
- **Creator / Publisher** — standard CCVEI flow.
- **Loader** — imports published products into the active sequence.
- **Scene Inventory** — manages loaded products + versions.

## Auto-publishing workfile

`PremiereWorkfileCreator` is an `AutoCreator` with
`product_type = "workfile"` (not `workfileCompositing` — earlier KB
versions had this wrong). It runs on every reset, registering the
workfile instance alongside any artist-marked publishes.

## What you can load

Two loaders ship:

| Loader | File | Scope |
|--------|------|-------|
| `FileLoader` | `plugins/load/load_file.py` | Images, movies, audio. `product_base_types = {"*"}` — permissive; filtered by `IMAGE_EXTENSIONS + VIDEO_EXTENSIONS` (and audio exts). |
| `AECompLoader` | `plugins/load/load_aftereffects_comp.py` | Loads `.aep` files. `product_base_types = {"workfile"}`. |

The artist-facing help article summarises loadable types as
"*image, plate, render, prerender, review, audio*", but that's a
human-readable simplification — the code is permissive over any
product whose representation's extension matches the image/video/audio
sets.

**Loaded images must stay as smart layers** to remain updatable.
Rasterising breaks the version link; Scene Inventory will no longer
be able to update the layer.

## Version management

Scene Inventory supports the standard actions (Update to Latest,
Change to Hero, Set Version, Switch Asset, Remove, Download/Upload).

## Metadata storage

Integration metadata is stored inside the `.prproj` — the
`AYON Metadata - DO NOT DELETE` bin is referenced in loader docstrings
as the location for round-tripping instance state. Don't rename or
delete it.

## Addon layout (source-confirmed)

```
ayon-premiere/
├─ package.py                    # name=premiere, app_host_name=premiere
├─ server/
│  ├─ __init__.py                # BaseServerAddon subclass (auto-template
│  │                             #   name; settings_model=PremiereSettings)
│  └─ settings.py                # PremiereSettings — single field
└─ client/ayon_premiere/
   ├─ __init__.py                # exports PremiereAddon
   ├─ addon.py                   # PremiereAddon(AYONAddon, IHostAddon)
   ├─ api/
   │  ├─ extension.zxp           # packaged CEP panel
   │  └─ extension/
   │     ├─ index.html           # panel HTML
   │     ├─ js/main.js           # panel JS entry
   │     ├─ jsx/hostscript.jsx   # ExtendScript bridge to Premiere
   │     └─ CSXS/manifest.xml    # ExtensionBundleId=io.ynput.PPRO.panel
   │                             # ExtensionBundleVersion=1.1.10 (current)
   ├─ plugins/
   │  ├─ create/
   │  │  └─ workfile_creator.py  # PremiereWorkfileCreator (AutoCreator)
   │  ├─ load/
   │  │  ├─ load_file.py         # FileLoader
   │  │  └─ load_aftereffects_comp.py  # AECompLoader
   │  └─ publish/
   │     ├─ collect_current_file.py    # CollectCurrentFile
   │     ├─ collect_workfile.py
   │     └─ increment_workfile.py
   └─ hooks/
      ├─ pre_launch_install_ayon_extension.py  # InstallAyonExtensionToPremiere
      └─ pre_launch_args.py
```

**Not present**: `plugins/inventory/` dir, `startup/` dir, `PremiereHost`
class.

## Settings (source-confirmed)

`server/settings.py` has exactly **one** top-level field:

| Field | Type | Default | Title |
|-------|------|---------|-------|
| `auto_install_extension` | `bool` | `True` | "Install AYON Extension" |

No nested submodels, no publish-profile fields, no product-name
templates. Accessed in code as `project_settings["premiere"]["auto_install_extension"]`.

## Caveats

- **Panel bundle ID is `io.ynput.PPRO.panel`.** Older KB versions
  documented `PPRS` — that's wrong.
- **Version bump discipline**: when updating the addon, bump
  `ExtensionBundleVersion` in `manifest.xml`. CEP caches by bundle
  version; bumping forces Premiere to pick up the new panel. The
  install hook also compares versions to decide when to replace.
- **CEP vs UXP**: Premiere still uses CEP. Newer Adobe apps are moving
  to UXP; if/when an Ayon UXP panel ships, install paths + APIs
  differ.
- **No Python-side host class**: don't look for `PremiereHost` — the
  CEP panel + WebSocket bridge play that role.
- **Smart-layer requirement** (per help docs, not enforced in code):
  train artists never to rasterise Ayon-loaded footage.

## Deeper reading

- `help.ayon.app/en/articles/3607676-working-with-premiere-in-ayon`
- `help.ayon.app/en/articles/1663762-premiere-addon-settings`

## Sources

- <https://help.ayon.app/en/articles/3607676-working-with-premiere-in-ayon>
- <https://help.ayon.app/en/articles/1663762-premiere-addon-settings>
- <https://github.com/ynput/ayon-premiere>
- <https://github.com/ynput/ayon-premiere/blob/develop/server/settings.py> (source-confirmed settings surface)
- <https://github.com/ynput/ayon-premiere/blob/develop/client/ayon_premiere/addon.py> (PremiereAddon class + interfaces)
- <https://github.com/ynput/ayon-premiere/blob/develop/client/ayon_premiere/api/extension/CSXS/manifest.xml> (io.ynput.PPRO.panel)
- <https://github.com/ynput/ayon-premiere/blob/develop/client/ayon_premiere/plugins/create/workfile_creator.py> (workfile product_type)
- <https://github.com/ynput/ayon-premiere/blob/develop/client/ayon_premiere/plugins/load/load_file.py> (FileLoader)
- <https://github.com/ynput/ayon-premiere/blob/develop/client/ayon_premiere/hooks/pre_launch_install_ayon_extension.py> (auto-install hook)
