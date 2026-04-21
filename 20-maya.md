# 20 — Autodesk Maya in Ayon

The most feature-complete DCC integration. Repo:
`github.com/ynput/ayon-maya` (100% Python, CI via
`ynput/ayon-addon-template`).

## AYON menu

Appears when Maya is launched from the Ayon launcher. Current context
(folder path + task name) shown at the top. Grouped sections:

**Global tools**: Creator/Publisher · Loader · Scene Inventory · Workfiles · Look Assigner

**Workfile actions**: Set Frame Range · Set Resolution · Set Colorspace · Apply All · Workfile Builder

**Pipeline tools**: Experimental · (admin sub-menu)

## The golden rule

> **AYON Maya is data-driven.** Publishing behaviour depends on what's in
> the scene. Artists create named instances (object sets) to mark what
> should be published; validators enforce the rest.

## Product types

### Model

- Geometry named with a type suffix (`sphere_GEO`)
- Frozen transforms (validator catches non-identity transforms)
- Wrapped in a parent group (`model_GRP`)
- Create → Model; `out_SET` holds publishable geometry

**Outputs (representations)**: Alembic, Maya Scene (`.ma`), OBJ

**Load as**: Alembic, Maya ASCII, V-Ray Proxy, Arnold Standin, GPU
Cache, USD

### Look

Shader + assignment + attribute bundle.
1. Load a Model.
2. Assign shaders, textures, attrs.
3. Create → Look.
4. Publish.

Looks are applied via the **Look Assigner** tool (`AYON → Look
Assigner`). Two-panel UI (models | looks). Steps: activate "load all
products" → pick model → right-click look → Assign. The tool auto-loads
the latest look.

**Arnold caveat**: Looks applied to Arnold Standins support only a
limited attribute set (visibility, shadow, subdivision) through Arnold
operators.

**V-Ray Proxy caveat**: `vrmesh` can't hold looks — it lacks the IDs
needed to map shaders to geometry. Ayon publishes V-Ray Proxies with a
paired Alembic so the Loader uses the ABC instead for look-capable
loads.

### Rig

Hierarchy convention with four required object sets (example variant
`Main`):

| Set | Purpose |
|-----|---------|
| `rigMain_controls_SET` | Control objects artists animate |
| `rigMain_out_SET` | Deformed geometry at publish |
| `rigMain_skeletonMesh_SET` | Static skeleton + mesh for FBX export |
| `rigMain_skeletonAnim_SET` | Bone hierarchy driving animation |

Validators enforce unique `cbId` values, hidden joints (per studio
convention), and that controller attributes reset to defaults.

### Animation

Auto-created when a Rig is loaded. Publish → baked geometry in Alembic
(default). Alternate outputs configurable per project.

### Pointcache

1. Build or load a rig.
2. Animate.
3. Create → Pointcache. Populate the `_SET` with the animated geometry.
   Include a **`proxy` object set** for viewport-friendly representation.
4. Set frame range, handles, step in the instance attrs.
5. Publish → Alembic.

**Alembic gotcha**: `writeNormals` is enabled by default, which **locks
normals** on reference. Downstream deformation may misbehave. If
downstream needs soft normals, turn it off on the instance.

### Camera

Create → Camera on a camera node. Publish → Alembic (clean, baked).

### Set Dress / Layout

Load published assets, position them, select all, create Set Dress or
Layout. Publish produces a JSON-ish representation that recreates the
layout on load.

**Multi-shot Layout** — automates creation across shots via the Camera
Sequencer. One publish, many shot-specific outputs.

### Arnold Scene Source (`ass`)

- Contains selected geometry exported as a single `.ass` or a per-frame
  sequence.
- Renderable **standins** at load time.

**Arnold Scene Source Proxy**: same + a `proxy_SET` for viewport.
Requirement: "content and proxy nodes need to share the same names
(including shape names **and** cbIds)."

Validators check:
- Hierarchy (model groups nested in standin sets)
- Texture paths (relative)
- `cbId` presence

Repair actions fix common issues.

### V-Ray Proxy

- Select geometry → Create → V-Ray Proxy.
- Publish produces **both** `.vrmesh` and `.abc`.
- Loader prefers the ABC if present (flexibility + look support).

### Redshift Proxy

- Create → Redshift Proxy.
- Publish → `.rs` proxy (animation: per-frame `.rs` files).
- Load attaches Redshift Proxy parameters to the mesh.

### Render

1. `AYON → Set Frame Range`
2. Create → Render (per render layer)
3. Configure render settings — Ayon sets the image prefix
   automatically per renderer (see below)
4. Assign a renderable camera to each render layer

**Image prefix defaults per renderer** (from
`server/settings/render_settings.py` → `DEFAULT_RENDER_SETTINGS`):

| Renderer | Default image prefix |
|----------|---------------------|
| Arnold | `<Scene>/<RenderLayer>/<RenderLayer>` |
| V-Ray | `<scene>/<Layer>/<Layer>` (lowercase placeholders) |
| Redshift | `<Scene>/<RenderLayer>/<RenderLayer>` |
| RenderMan | `<layer>{aov_separator}<aov>.<f4>.<ext>` |
| Maya Hardware 2.0 | `<Scene>/<RenderLayer>/<RenderLayer>` |

These prefixes are **relative** — the `maya/...` parent directory that
artists often see comes from the **project anatomy** (workfile / publish
template roots), not from the renderer prefix. AOV suffixes (`_<AOV>`)
are applied separately by render-setup code, not hard-coded in the
default prefix.

**Validators** enforce:
- Default camera set
- Animation enabled in render settings
- Prefix contains required tokens

**Publishing to Deadline**:
- Render job → image-sequence publishing job → preview quicktime job
  (beauty passes only).
- See `24-deadline.md` / skill `ayon-deadline`.

**Tile rendering**: enable on the render instance. Deadline generates
per-frame tile jobs, assembly jobs, then publishes.

**Render Setup Presets**: serialise current render settings to JSON;
distribute + load into other scenes.

### Review (playblast)

- Create → Review on a camera. Optionally parent model geometry under
  the review set.
- Publish: playblast → FFmpeg transcode → burnin insertion → ftrack
  upload (if ftrack addon is active).

### Workfile

Auto-published as part of most publishes (see `AutoCreator` pattern in
`08-publishing.md`). Ensures the source `.ma` / `.mb` is preserved
alongside outputs.

## Scene hygiene — validator hit-list

- **Freeze transforms** on geometry
- **Unique naming** per studio convention
- Correct **set membership** (`out_SET`, `controls_SET`, `proxy`, …)
- **`cbId`** — color-buddy-id — unique per object, used for reliable
  cross-context linking (Connect Geometry action, look assignment,
  Arnold Proxy pairing)
- **Hidden joints** in rigs
- **Controller attributes reset to defaults** before publish

## Inventory actions

- **Connect Geometry** — links geometry between animation/pointcache
  containers and other containers via blendshape, matched by `cbId`.

Custom inventory actions via `InventoryAction` — see
`15-loaders-inventory.md`.

## Related integrations

| Article | URL |
|---------|-----|
| Multiverse | <https://help.ayon.app/en/articles/7718098-working-with-multiverse-in-ayon> |
| Yeti | <https://help.ayon.app/en/articles/4740816-working-with-yeti-in-ayon> |
| XGen | <https://help.ayon.app/en/articles/7485931-working-with-xgen-in-ayon> |
| Ornatrix | <https://help.ayon.app/en/articles/7811185-working-with-ornatrix-in-ayon> |
| Arnold | <https://help.ayon.app/en/articles/2888902-working-with-arnold-in-ayon> |
| V-Ray | <https://help.ayon.app/en/articles/3842653-working-with-vray-in-ayon> |
| Redshift | <https://help.ayon.app/en/articles/0021176-working-with-redshift-in-ayon> |

## Key classes (source-confirmed)

- **Server**: `MayaAddon(BaseServerAddon)` with `MayaSettings`
  (`server/__init__.py`, `server/settings/main.py`).
- **Client addon**: `MayaAddon(AYONAddon, IHostAddon)` in
  `client/ayon_maya/addon.py`.
- **Host**: `MayaHost(HostBase, IWorkfileHost, ILoadHost, IPublishHost)`
  in `client/ayon_maya/api/pipeline.py`.
- **`cbId` attribute**: string attribute on DependencyNodes; read via
  `get_id(node)` and written via `set_id(node, unique_id, overwrite=False)`
  in `client/ayon_maya/api/lib.py`. Batch generator: `generate_ids(nodes)`.
- **Connect Geometry**: `ConnectGeometry(InventoryAction)` in
  `client/ayon_maya/plugins/inventory/connect_geometry.py`.
- **Look Assigner UI**: `MayaLookAssignerWindow` in
  `client/ayon_maya/tools/mayalookassigner/app.py` (the tool is named
  "Look Assigner" — never "Look Manager").
- **Creator base classes**: `MayaCreator`, `MayaHiddenCreator`,
  `RenderlayerCreator` in `client/ayon_maya/api/plugin.py`.

Representative creators (all in `client/ayon_maya/plugins/create/`):
`CreateModel`, `CreateLook`, `CreateRig`, `CreateAnimation` + pointcache
(`create_animation_pointcache.py`), `CreateCamera`, `CreateSetDress`,
`CreateLayout`, `CreateArnoldSceneSource` + `CreateArnoldSceneSourceProxy`
(same file), `CreateVrayProxy`, `CreateRedshiftProxy`, `CreateRender`,
`CreateReview`, `CreateWorkfile`. Additional: Yeti, XGen, Ornatrix,
Multiverse (USD/Look/USDComp/USDOver), USD Layer / USD / Maya USD Scene,
Matchmove, Unreal Skeletal/Static/YetiCache, V-Ray Scene, Multishot
Layout.

## Addon layout snapshot

```
ayon-maya/
└─ client/ayon_maya/
   ├─ api/            # MayaHost, lib, cbId, menu, tools, Look Assigner
   ├─ plugins/
   │  ├─ create/      # Creators per product type (~33 files)
   │  ├─ load/        # Loaders per representation (~27 files)
   │  ├─ publish/     # Collectors / validators / extractors / integrators (~90)
   │  └─ inventory/   # InventoryAction subclasses (e.g. ConnectGeometry)
   ├─ hooks/          # MayaPreAutoLoadPlugins, PreCopyMel,
   │                  # MayaPreOpenWorkfilePostInitialization
   ├─ startup/        # userSetup.py — installs MayaHost via install_host()
   │                  # loads plugins if explicit_plugins_loading, opens
   │                  # workfile from AYON_MAYA_WORKFILE_PATH
   ├─ tools/mayalookassigner/  # Look Assigner UI (MayaLookAssignerWindow)
   └─ vendor/
```

Browse `github.com/ynput/ayon-maya/client/ayon_maya/` for current source
(plugin names drift version-to-version).

## Deeper reading

- `help.ayon.app/en/articles/6811988-working-with-maya-in-ayon` — artist guide
- `help.ayon.app/en/articles/6062888-look-assigner` — Look Assigner
- `help.ayon.app/en/collections/8127361-maya-addon-settings` — settings
  articles collection

## Sources

- <https://help.ayon.app/en/articles/6811988-working-with-maya-in-ayon>
- <https://help.ayon.app/en/articles/6062888-look-assigner>
- <https://help.ayon.app/en/articles/2888902-working-with-arnold-in-ayon>
- <https://help.ayon.app/en/articles/3842653-working-with-vray-in-ayon>
- <https://help.ayon.app/en/articles/0021176-working-with-redshift-in-ayon>
- <https://github.com/ynput/ayon-maya>
- <https://github.com/ynput/ayon-maya/blob/develop/client/ayon_maya/api/pipeline.py> (MayaHost)
- <https://github.com/ynput/ayon-maya/blob/develop/client/ayon_maya/api/lib.py> (cbId: `get_id` / `set_id` / `generate_ids`)
- <https://github.com/ynput/ayon-maya/blob/develop/client/ayon_maya/plugins/create/create_rig.py> (rig object-set naming)
- <https://github.com/ynput/ayon-maya/blob/develop/client/ayon_maya/plugins/inventory/connect_geometry.py>
- <https://github.com/ynput/ayon-maya/blob/develop/client/ayon_maya/tools/mayalookassigner/app.py> (`MayaLookAssignerWindow`)
- <https://github.com/ynput/ayon-maya/blob/develop/server/settings/render_settings.py> (image-prefix defaults)
