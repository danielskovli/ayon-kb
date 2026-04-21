# 21 — SideFX Houdini in Ayon

Repo: `github.com/ynput/ayon-houdini` (100% Python). Houdini is the most
USD-native DCC in Ayon — much of the USD pipeline is first-class here.

## Supported versions

Houdini **19.5+**.

## AYON menu

Appears when Houdini launches from the Ayon launcher. Current context
(folder path + task) shown at the top.

**Global tools**: Creator/Publisher · Loader · Scene Inventory · Workfiles

**Workfile actions**:
- **Set Frame Range** — pulls from folder's attribs
- **Update Houdini Vars** — refreshes the Houdini env vars derived from
  context (`$HIP`, studio vars)
- **Template Builder** — build workfiles from pre-made templates

**Pipeline tools**: Experimental tools sub-menu.

## Publishing

Select relevant nodes → **AYON → Create** → pick product type → toggle
**Use Selection** if appropriate → set variant → confirm.

Ayon automatically creates the matching **ROPs in `/out`** with paths
and frame ranges pre-configured.

Publisher UI is the standard CCVEI flow (see `08-publishing.md`).

## Product types

### Geometry & caching

| Type | Output formats |
|------|---------------|
| **PointCache** | Alembic, Bgeo |
| **VDB Cache** | OpenVDB |
| **Static Mesh** | FBX |
| **Houdini Digital Asset (HDA)** | `.hda` / `.otl` |

### Rendering

| Type | Notes |
|------|-------|
| **Arnold Scene Source** | `.ass` archive |
| **Arnold ROP** | Render directly |
| **Mantra ROP** | Legacy renderer |
| **Karma ROP** | Solaris / Karma |
| **Redshift ROP** | Render |
| **Redshift Proxy** | Proxy geometry |
| **V-Ray ROP** | Render |

### Other

| Type | Notes |
|------|-------|
| **Camera** | Alembic |
| **Composite** | Image sequence |
| **Review** | Flipbook / render for review |
| **USD** | Generic USD publish |
| **USD Assembly** | USD assembly structure |
| **USD Groom** | Groom-specific USD |
| **USD Look** | Look USD |
| **USD Model** | Model USD |
| **USD Render** | Render USD |

The USD variants form Ayon's **USD contribution workflow** — assets
layered into shot stages. See Ayon's USD help collection.

## Deadline render targets

Five options when a render ROP is about to submit:

| Target | Behaviour |
|--------|-----------|
| **Local Machine** | Render locally |
| **Use existing frames (local)** | Skip render; publish from existing frames on disk |
| **Farm Rendering** | Submit render + publish jobs to Deadline |
| **Farm Rendering with split export/render jobs** | Export intermediate (`.ass`/`.ifd`) → render → publish (3 jobs). Isolates Houdini license usage |
| **Local Export & Farm Rendering** | Export locally, render on farm |

See `24-deadline.md` / skill `ayon-deadline`.

## Loading — the HDA system

Ayon ships loader HDAs under `client/ayon_houdini/startup/otls/`
(source-confirmed):

| HDA path | Variants | Python loader class (file) | Purpose |
|----------|----------|----------------------------|---------|
| `ayon.generic_loader.hda/` | Lop, Object (OBJ), Sop | `FilePathLoader` (`plugins/load/load_filepath.py`) | **Generic Loader** — any product as filepath |
| `ayon_lop_import.hda/` | Lop | `LOPLoadAssetLoader` (`plugins/load/load_asset_lop.py`) | **LOP Import** — USD assets in LOP |
| `ayon_lop_load_shot.hda/` | Lop | `LOPLoadShotLoader` (`plugins/load/load_shot_lop.py`) | **Load Shot** — USD shots as sublayers |
| `ayon_lop_mute_layers.hda/` | Lop | — | Utility — mute USD layers |

Additional non-HDA loaders: `HdaLoader` (`load_hda.py`) for HDA
references, plus format-specific loaders (Alembic, ASS, Bgeo, Camera,
FBX, Image, VDB, Redshift Proxy, USD sublayer/reference/SOP, Layout).

Each HDA bundles a `PythonModule` backing script, a `DialogScript`
parameter layout, and `OnCreated`/`OnLoaded`/`OnDeleted` callbacks.

### 1. Generic Loader (`FilePathLoader` + `ayon.generic_loader.hda`)

"Loads any AYON product as a filepath." Works across OBJ / SOP / LOP
contexts.

- Parameter-driven: pick project → folder → product → version →
  representation.
- The `File` parameter auto-populates — copy/paste as needed.
- `$F` (with padding, e.g. `$F4`) and `<UDIM>` tokens auto-applied in
  `client/ayon_houdini/api/hda_utils.py` via `get_filepath_from_context()`.
- Optional **AYON Entity URI mode** (`use_ayon_entity_uri` parameter)
  stores the path as an Ayon URI, resolved at cook time via
  `_construct_ayon_entity_uri_str()` and `_resolve_entity_uri()`.

### 2. LOP Import (`LOPLoadAssetLoader` + `ayon_lop_import.hda`)

Purpose-built for loading **USD assets** into LOP contexts.

### 3. Load Shot (`LOPLoadShotLoader` + `ayon_lop_load_shot.hda`)

Loads **USD shots as sublayers** on the current stage.

### 4. Mute Layers (utility HDA `ayon_lop_mute_layers.hda`)

Utility for muting USD layers. Not a loader — no Python loader class.

### Updating & refreshing

- **Clear Cache** button — refreshes the node's cached data.
- **Reload Files** — reloads contents of all imported files.
- **Nodes Referencing File** — shows which other nodes reference this
  loader's output (useful for refactoring).
- **Thumbnail display** — optional, with a manual update button.

When loading via any method, content wraps in an **`AYON_CONTAINERS`
subnetwork** so Scene Inventory can track it.

## Scene Inventory

Same UI as every other DCC. Houdini's HDAs integrate so that **Update
to Latest** / **Switch Asset** propagate through the cached file
parameter.

## Key classes (source-confirmed)

- Server: `Houdini(BaseServerAddon)` (`server/__init__.py`)
- Client: `HoudiniAddon(AYONAddon, IHostAddon)` — workfile extensions
  `[".hip", ".hiplc", ".hipnc"]` (`client/ayon_houdini/addon.py`)
- Host: `HoudiniHost(HostBase, IWorkfileHost, ILoadHost, IPublishHost)`
  in `client/ayon_houdini/api/pipeline.py`
- HDA utilities: `client/ayon_houdini/api/hda_utils.py` (token apply,
  Entity URI resolution, cache utilities)
- ROP creator base: `RenderLegacyProductTypeCreator` — every ROP creator
  exposes the 5 Deadline render targets (local, local_no_render, farm,
  farm_split, local_export_farm_render)

## Addon layout

```
ayon-houdini/
└─ client/ayon_houdini/
   ├─ api/            # HoudiniHost, lib, hda_utils, pipeline, workfile_template_builder
   ├─ plugins/
   │  ├─ create/      # ~24 creators (+ `convert_legacy.py`)
   │  ├─ load/        # ~20 loaders (HDA-backed + format-specific)
   │  ├─ publish/     # ~74 collectors / validators / extractors
   │  └─ inventory/   # select_containers, show_parameters, set_camera_resolution
   ├─ hooks/
   │  ├─ set_paths.py                    # SetPath — HOUDINI_PATH, HFS, OCIO
   │  └─ set_default_display_and_view.py # SetDefaultDisplayView — viewport defaults
   └─ startup/
      ├─ MainMenuCommon.xml              # AYON menu
      ├─ OPmenu.xml, PARMmenu.xml        # context menus
      ├─ otls/                           # HDAs (see Loader HDAs above)
      ├─ husdplugins/                    # Houdini USD plugin extensions
      └─ python3.{7,9,10,11}libs/        # per-Python-version startup
```

Check `github.com/ynput/ayon-houdini/client/ayon_houdini/` for current
plugin/HDA names.

## Tips / traps

- **Workfile vars** — always run **Update Houdini Vars** after changing
  context, or `$HIP` / `$JOB` / studio vars will lag behind.
- **USD pipeline** — Houdini is the primary USD-producing DCC; learn
  the `USD Assembly` / `USD Groom` / `USD Look` / `USD Model` /
  `USD Render` variants before writing custom creators.
- **Split export/render** — use this when Houdini license count is
  tight; the render-only job uses the Arnold/Mantra plugin on
  Deadline instead of Houdini itself.
- **HDAs as the loader surface** — new file format? Prefer extending
  the Generic Loader HDA over writing a pure-Python `LoaderPlugin`.
  HDAs give artists parameter UI for free.

## Deeper reading

- `help.ayon.app/en/articles/2777507-working-with-houdini-in-ayon`
- `help.ayon.app/en/articles/4210042-houdini-ayon-loader-hdas`
- `help.ayon.app/en/articles/0982414-configure-houdini-addon`
- USD pipeline: `help.ayon.app/en/articles/6167506-why-and-what-is-openusd`
  + `help.ayon.app/en/collections/4882924-ayon-openusd-contribution-workflow`

## Sources

- <https://help.ayon.app/en/articles/2777507-working-with-houdini-in-ayon>
- <https://help.ayon.app/en/articles/4210042-houdini-ayon-loader-hdas>
- <https://github.com/ynput/ayon-houdini>
- <https://github.com/ynput/ayon-houdini/blob/develop/client/ayon_houdini/api/pipeline.py> (`HoudiniHost`, `get_or_create_ayon_container`)
- <https://github.com/ynput/ayon-houdini/blob/develop/client/ayon_houdini/api/hda_utils.py> (`$F` / `<UDIM>` token apply, Entity URI mode)
- <https://github.com/ynput/ayon-houdini/tree/develop/client/ayon_houdini/startup/otls> (HDAs: generic_loader, lop_import, lop_load_shot, lop_mute_layers)
- <https://github.com/ynput/ayon-houdini/blob/develop/client/ayon_houdini/plugins/load/load_filepath.py> (`FilePathLoader`)
- <https://github.com/ynput/ayon-houdini/blob/develop/client/ayon_houdini/plugins/create/create_arnold_rop.py> (5 render-target pattern)
