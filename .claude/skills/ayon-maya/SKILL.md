---
name: ayon-maya
description: Autodesk Maya in Ayon (`ayon-maya`, Maya publish). Use when writing Maya creators/loaders/validators, debugging a Maya publish, or helping an artist with Maya-in-Ayon. Data-driven publishing via object-set conventions (`_SET`, `out_SET`, `controls_SET`, `skeletonMesh_SET`, `skeletonAnim_SET`, `proxy_SET`, `model_GRP`, `rigMain_controls_SET`, `rigMain_out_SET`). Product catalogue: Model, Look, Rig, Animation, Pointcache, Camera, Set Dress, Layout, Arnold Scene Source + Proxy (Arnold standin), V-Ray Proxy, Redshift Proxy, Render, Review. `cbId` identifiers. Look Assigner / Look Manager workflow. Render prefix convention, render setup presets, tile rendering. Scene-hygiene validators (freeze transforms, writeNormals). Inventory actions (Connect Geometry). Related plugins: XGen, Yeti, Multiverse, Ornatrix. Renderers: Arnold, V-Ray, Redshift.
---

# Autodesk Maya in Ayon

Repo: `github.com/ynput/ayon-maya` (100% Python). Feature-complete
reference integration.

## AYON menu

Global tools: Creator/Publisher · Loader · Scene Inventory · Workfiles
· Look Assigner.

Workfile actions: Set Frame Range · Set Resolution · Set Colorspace ·
Apply All · Workfile Builder.

## Golden rule

**Data-driven.** Publish behaviour depends on what's in the scene.
Artists create named instances (object sets); validators enforce the rest.

## Product types

### Model
- Geometry named `_GEO` suffix; frozen transforms; wrapped in `model_GRP`.
- Create → Model; `out_SET` holds publishable geo.
- Outputs: Alembic / Maya Scene / OBJ.
- Load as: Alembic / `.ma` / V-Ray Proxy / Arnold Standin / GPU
  Cache / USD.

### Look
- Shader + assignment + attribute bundle.
- Flow: load Model → assign shaders/textures/attrs → Create → Look →
  Publish.
- Apply via **Look Assigner**: 2-panel UI (models | looks). Activate
  "load all products" → pick model → right-click look → Assign.
  Auto-loads the **latest look**.
- **Arnold Standin** applies only visibility / shadow / subdivision
  through operators.
- **V-Ray `.vrmesh`** can't hold looks — Ayon publishes ABC alongside
  and the Loader prefers ABC.

### Rig
Four required object sets (variant name = `Main` in examples):

| Set | Purpose |
|-----|---------|
| `rigMain_controls_SET` | Animation controls |
| `rigMain_out_SET` | Deformed geometry at publish |
| `rigMain_skeletonMesh_SET` | Static skeleton + mesh (FBX export) |
| `rigMain_skeletonAnim_SET` | Bone hierarchy driving animation |

Validators: unique `cbId`, hidden joints, controller attrs reset to
defaults.

### Animation
Auto-created on Rig load. Publish → baked Alembic (default).

### Pointcache
- `_SET` with animated geo; `proxy` set for viewport.
- Frame range, handles, step on instance.
- Publish → Alembic.
- **Trap**: `writeNormals` enabled by default locks normals on
  reference; disable if downstream needs soft normals.

### Camera
Create → Camera. Publish → Alembic (clean, baked).

### Set Dress / Layout
Load assets → position → select → Create Set Dress or Layout.
Multi-shot Layout automates creation across shots via Camera Sequencer.

### Arnold Scene Source (`ass`)
Exports selected geo to `.ass` (single or per-frame sequence).
**Arnold Scene Source Proxy**: pair content + `proxy_SET`. Content and
proxy nodes must share names + `cbId`s. Validators: hierarchy, relative
texture paths, `cbId`.

### V-Ray Proxy
Publish produces **both** `.vrmesh` and `.abc`. Loader prefers ABC
(more flexible, supports looks).

### Redshift Proxy
Publish → `.rs` (animation: per-frame). Load attaches RS Proxy params
to the mesh.

### Render
- `AYON → Set Frame Range`.
- Create → Render (per layer).
- Image prefix is set per renderer by Ayon (defaults in
  `server/settings/render_settings.py` → `DEFAULT_RENDER_SETTINGS`):
  - Arnold / Redshift / Maya Hardware: `<Scene>/<RenderLayer>/<RenderLayer>`
  - V-Ray: `<scene>/<Layer>/<Layer>` (lowercase)
  - RenderMan: `<layer>{aov_separator}<aov>.<f4>.<ext>`
  - The `maya/...` parent dir comes from **project anatomy**, not the
    renderer prefix. AOV suffixes are applied by render-setup code, not
    hard-coded in the prefix.
- Assign renderable camera per layer.
- Validators: default cam set, animation enabled, prefix tokens valid.
- Deadline: render job → image-sequence publish → preview QT (beauty
  only).

**Tile rendering**: enable on instance → Deadline generates per-frame
tile jobs, assembly jobs, then publishes.

**Render Setup Presets**: serialise render settings to JSON;
distribute + load into other scenes.

### Review (playblast)
Create → Review on a camera. Publish → playblast → FFmpeg → burnin →
ftrack upload.

### Workfile
Auto-published alongside every publish (`AutoCreator`).

## Scene hygiene — validator hit-list

- Freeze transforms on geometry
- Unique naming per studio convention
- Correct set membership (`out_SET`, `controls_SET`, `proxy`, …)
- `cbId` present (color-buddy-id — unique per object; enables reliable
  cross-context linking)
- Hidden joints on rigs
- Controller attrs reset to defaults

## Inventory actions

- **Connect Geometry** — links anim/pointcache containers to other
  containers via blendshape, matched by `cbId`.
- Add custom actions via `InventoryAction` — see `ayon-loaders-inventory`.

## Key classes (source-confirmed)

- Server: `MayaAddon(BaseServerAddon)` (`server/__init__.py`)
- Client: `MayaAddon(AYONAddon, IHostAddon)` (`client/ayon_maya/addon.py`)
- Host: `MayaHost(HostBase, IWorkfileHost, ILoadHost, IPublishHost)`
  (`client/ayon_maya/api/pipeline.py`)
- `cbId` read/write: `get_id(node)`, `set_id(node, uid, overwrite=False)`,
  `generate_ids(nodes)` in `client/ayon_maya/api/lib.py`
- Connect Geometry: `ConnectGeometry(InventoryAction)` in
  `client/ayon_maya/plugins/inventory/connect_geometry.py`
- Look Assigner UI: `MayaLookAssignerWindow` in
  `client/ayon_maya/tools/mayalookassigner/app.py` — tool is "Look
  Assigner" (never "Look Manager").
- Creator bases: `MayaCreator`, `MayaHiddenCreator`, `RenderlayerCreator`
  in `client/ayon_maya/api/plugin.py`.

## Addon layout

```
ayon-maya/client/ayon_maya/
├─ api/            # MayaHost, lib, cbId, menu, render tools, Look Assigner
├─ plugins/
│  ├─ create/      # ~33 creators per product type
│  ├─ load/        # ~27 loaders per representation
│  ├─ publish/     # ~90 collectors / validators / extractors / integrators
│  └─ inventory/   # InventoryActions (e.g. ConnectGeometry)
├─ hooks/          # MayaPreAutoLoadPlugins, PreCopyMel, MayaPreOpenWorkfilePostInitialization
├─ startup/        # userSetup.py — installs MayaHost
├─ tools/mayalookassigner/  # Look Assigner window
└─ vendor/
```

## Related integrations (brief)

| Tool | Notes |
|------|-------|
| Multiverse | Separate "Working with Multiverse" article |
| Yeti | Groom product type; see Yeti article |
| XGen | Grooming; see XGen article |
| Ornatrix | Grooming; see Ornatrix article |
| Arnold | Standins, Scene Source; look support via operators |
| V-Ray | `.vrmesh` + paired `.abc`; no look on raw vrmesh |
| Redshift | Proxy product; per-frame animation |

## Deeper reading

- `20-maya.md` — full file with Sources
- `help.ayon.app/en/collections/8127361-maya-addon-settings` — settings
- Skill `ayon-publishing` — Creator/Validator patterns
- Skill `ayon-loaders-inventory` — LoaderPlugin base

## Sources

- <https://help.ayon.app/en/articles/6811988-working-with-maya-in-ayon>
- <https://help.ayon.app/en/articles/6062888-look-assigner>
- <https://help.ayon.app/en/articles/2888902-working-with-arnold-in-ayon>
- <https://help.ayon.app/en/articles/3842653-working-with-vray-in-ayon>
- <https://help.ayon.app/en/articles/0021176-working-with-redshift-in-ayon>
- <https://github.com/ynput/ayon-maya>
- <https://github.com/ynput/ayon-maya/blob/develop/client/ayon_maya/api/pipeline.py> (MayaHost)
- <https://github.com/ynput/ayon-maya/blob/develop/client/ayon_maya/api/lib.py> (cbId helpers)
- <https://github.com/ynput/ayon-maya/blob/develop/client/ayon_maya/plugins/inventory/connect_geometry.py>
- <https://github.com/ynput/ayon-maya/blob/develop/client/ayon_maya/tools/mayalookassigner/app.py> (`MayaLookAssignerWindow`)
- <https://github.com/ynput/ayon-maya/blob/develop/server/settings/render_settings.py> (image prefix defaults)
