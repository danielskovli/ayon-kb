---
name: ayon-houdini
description: SideFX Houdini in Ayon (`ayon-houdini`, Houdini 19.5+). Use when writing Houdini creators/loaders, debugging a Houdini publish, or doing USD contribution workflow. Addon menu: Set Frame Range, Update Houdini Vars, Template Builder. Publishing via automatic `/out` ROPs — product catalogue: PointCache, VDB, Static Mesh, HDA, Arnold/Mantra/Karma ROP/Redshift Proxy (Houdini)/V-Ray ROPs, Camera, Composite, Review, USD variants (USD Assembly, USD Groom, USD Look, USD Model, USD Render). Loader HDAs: Generic Loader HDA, LOP Import, Load Shot — with `$F` and `<UDIM>` auto-tokens and AYON Entity URI mode. Containers wrapped in `AYON_CONTAINERS` / `AYON_CONTAINER` subnetwork. Uses `$HIP`, `$JOB`. Five Deadline render targets: Local / Use existing / Farm / Farm split export render / Local Export & Farm.
---

# SideFX Houdini in Ayon

Repo: `github.com/ynput/ayon-houdini`. The most USD-native DCC in the
Ayon stack.

## Supported versions

Houdini **19.5+**.

## AYON menu

Global tools: Creator/Publisher · Loader · Scene Inventory · Workfiles.

Workfile actions:
- **Set Frame Range** — from folder attribs
- **Update Houdini Vars** — refresh `$HIP`/`$JOB`/studio vars from
  context
- **Template Builder** — instantiate workfiles from templates

## Publishing

Select nodes → **AYON → Create** → product type → toggle **Use
Selection** → variant → confirm. Ayon creates matching **ROPs in
`/out`** with paths + frame ranges pre-wired. Standard CCVEI flow from
there.

## Product types

### Geometry & caching
| Type | Outputs |
|------|---------|
| PointCache | Alembic · Bgeo |
| VDB Cache | OpenVDB |
| Static Mesh | FBX |
| Houdini Digital Asset (HDA) | `.hda` / `.otl` |

### Rendering
| Type | Notes |
|------|-------|
| Arnold Scene Source | `.ass` archive |
| Arnold ROP | Render direct |
| Mantra ROP | Legacy |
| Karma ROP | Solaris / Karma |
| Redshift ROP | Render |
| Redshift Proxy | Proxy geo |
| V-Ray ROP | Render |

### Other
Camera (Alembic) · Composite (image seq) · Review · USD · USD
Assembly · USD Groom · USD Look · USD Model · USD Render.

USD variants form Ayon's **USD contribution workflow** — assets layered
into shot stages.

## Deadline render targets

Five options at submission:

1. **Local Machine**
2. **Use existing frames (local)** — skip render; publish from existing
3. **Farm Rendering**
4. **Farm split export/render** — export (.ass/.ifd) → render →
   publish; **isolates Houdini licence usage**
5. **Local Export & Farm Rendering**

See `ayon-deadline`.

## Loader HDAs (source-confirmed)

Ayon ships four HDAs at `client/ayon_houdini/startup/otls/`:

| HDA | Python loader | Purpose |
|-----|---------------|---------|
| `ayon.generic_loader.hda` (Lop / OBJ / Sop variants) | `FilePathLoader` | **Generic Loader** — any product as filepath |
| `ayon_lop_import.hda` | `LOPLoadAssetLoader` | **LOP Import** — USD assets in LOP |
| `ayon_lop_load_shot.hda` | `LOPLoadShotLoader` | **Load Shot** — USD shots as sublayers |
| `ayon_lop_mute_layers.hda` | — | Utility — mute USD layers |

Additional: `HdaLoader` (`load_hda.py`) for HDA references, plus
format-specific loaders (Alembic, ASS, Bgeo, Camera, FBX, Image, VDB,
Redshift Proxy, USD sublayer/reference/SOP, Layout).

### Generic Loader
- Any product as a filepath; OBJ / SOP / LOP.
- Params: project → folder → product → version → representation.
- `File` parameter auto-populates.
- `$F` (with padding) + `<UDIM>` tokens auto-applied by
  `get_filepath_from_context()` in `api/hda_utils.py`.
- Optional **AYON Entity URI mode** (`use_ayon_entity_uri` param) —
  stores path as Ayon URI, resolved at cook time via
  `_construct_ayon_entity_uri_str()` / `_resolve_entity_uri()`.

### LOP Import
USD assets in LOP contexts.

### Load Shot
USD shots as sublayers on the current stage.

### Update & refresh
- **Clear Cache** — refresh cached data
- **Reload Files** — reload all imported files
- **Nodes Referencing File** — list dependents
- **Thumbnail display** — optional, manual update button

All loads wrap in an **`AYON_CONTAINERS`** subnet (at
`/obj/AYON_CONTAINERS`) created by `get_or_create_ayon_container()` in
`client/ayon_houdini/api/pipeline.py`. Legacy `/obj/AVALON_CONTAINERS`
is also recognised for backward compat.

## Key classes (source-confirmed)

- Server: `Houdini(BaseServerAddon)` (`server/__init__.py`)
- Client addon: `HoudiniAddon(AYONAddon, IHostAddon)` —
  extensions `[".hip", ".hiplc", ".hipnc"]` (`client/ayon_houdini/addon.py`)
- Host: `HoudiniHost(HostBase, IWorkfileHost, ILoadHost, IPublishHost)`
  (`client/ayon_houdini/api/pipeline.py`)
- HDA utils: `client/ayon_houdini/api/hda_utils.py` — token apply,
  entity URI, cache utilities
- ROP creator base: `RenderLegacyProductTypeCreator` — exposes the 5
  Deadline render targets on every ROP creator

## Addon layout

```
ayon-houdini/client/ayon_houdini/
├─ api/            # HoudiniHost, lib, hda_utils, pipeline, workfile_template_builder
├─ plugins/
│  ├─ create/      # ~24 creators
│  ├─ load/        # ~20 loaders
│  ├─ publish/     # ~74 collectors/validators/extractors
│  └─ inventory/   # select_containers, show_parameters, set_camera_resolution
├─ hooks/          # SetPath (HOUDINI_PATH/HFS/OCIO), SetDefaultDisplayView
└─ startup/
   ├─ MainMenuCommon.xml, OPmenu.xml, PARMmenu.xml
   ├─ otls/            # HDAs — see Loader HDAs
   ├─ husdplugins/
   └─ python3.{7,9,10,11}libs/
```

## Tips / traps

- **Always** run **Update Houdini Vars** after changing context, or
  `$HIP` / `$JOB` / studio vars will be stale.
- Houdini is the primary USD producer — learn the 5 USD variants
  before writing custom creators.
- **Split export/render** saves Houdini licences — the render-only
  stage uses Arnold/Mantra plugins on Deadline instead of Houdini.
- **Prefer extending the Generic Loader HDA** over writing a
  pure-Python `LoaderPlugin` — HDAs give artists parameter UI for
  free.

## Deeper reading

- `21-houdini.md` — full file with Sources
- `help.ayon.app/en/articles/0982414-configure-houdini-addon` — settings
- `help.ayon.app/en/articles/6167506-why-and-what-is-openusd` — USD
  overview
- `help.ayon.app/en/collections/4882924-ayon-openusd-contribution-workflow`
- Skill `ayon-deadline` — farm rendering
- Skill `ayon-publishing` — Creator patterns
- Skill `ayon-loaders-inventory` — LoaderPlugin base

## Sources

- <https://help.ayon.app/en/articles/2777507-working-with-houdini-in-ayon>
- <https://help.ayon.app/en/articles/4210042-houdini-ayon-loader-hdas>
- <https://github.com/ynput/ayon-houdini>
- <https://github.com/ynput/ayon-houdini/blob/develop/client/ayon_houdini/api/pipeline.py>
- <https://github.com/ynput/ayon-houdini/blob/develop/client/ayon_houdini/api/hda_utils.py>
- <https://github.com/ynput/ayon-houdini/tree/develop/client/ayon_houdini/startup/otls>
- <https://github.com/ynput/ayon-houdini/blob/develop/client/ayon_houdini/plugins/load/load_filepath.py>
- <https://github.com/ynput/ayon-houdini/blob/develop/client/ayon_houdini/plugins/create/create_arnold_rop.py>
