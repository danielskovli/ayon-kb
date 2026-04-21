# 08 — Publishing (pyblish + creators)

Ayon publishing is **pyblish** with a thin layer on top. The layer adds
Creators (explicit user-created instances with typed attributes), nicer error
reporting, and integration with the server data model.

## CCVEI pipeline

Pyblish runs plugins in ordered stages. Ayon uses all five:

| Stage | Order | Purpose |
|-------|-------|---------|
| **C**ollect | 0–0.49 | Gather data — scan scene, read metadata |
| **V**alidate | 0.5–0.99 | Check integrity — fail early with artist-readable errors |
| **E**xtract | 1.0–1.99 | Write output files to a staging dir |
| **I**ntegrate | 2.0+ | Upload + register with Ayon server (`ayon-python-api`) |

(Pyblish's own `CollectorOrder/ValidatorOrder/ExtractorOrder/IntegratorOrder`
constants give these numbers.)

A second "phase" sits **before** publishing:

- **Create** — artists open the Publisher UI, pick a Creator, mark what to
  publish. Produces `CreatedInstance` objects stored in scene metadata.

The full lifecycle is therefore: **Create → Collect → Validate → Extract →
Integrate**.

## CreatedInstance

Dict-like. Representative keys:

| Key | Kind | Notes |
|-----|------|-------|
| `id` | const | always `"pyblish.avalon.instance"` |
| `instance_id` | UUID | unique per instance |
| `family` / `productType` | str | e.g. `model`, `render` |
| `creator_identifier` | str | which Creator owns it |
| `creator_attributes` | dict | creator-defined bag |
| `variant` | str | e.g. `Main`, `High` |
| `productName` | str | computed from template |
| `active` | bool | toggled in UI |
| `folderPath` / `asset` | str | target folder |
| `task` | str | target task |

`CreatedInstance.data_to_store()` serialises to JSON for scene storage.

## CreateContext

Controller for the Create phase, lives in `ayon_core.pipeline.create`:

- Discovers Creator + publish plugins.
- Calls every Creator's `collect_instances()` to load existing scene data.
- Maintains an id → instance registry.
- Offers a shared cache between resets.
- Validates required host hooks (`get_context_data` / `update_context_data`).

Typical code:

```python
from ayon_core.pipeline import registered_host
from ayon_core.pipeline.create import CreateContext

host = registered_host()
ctx = CreateContext(host)
for inst in ctx.instances:
    print(inst.data["productName"])
```

## Creator base classes

All under `ayon_core.pipeline.create`:

### `BaseCreator`
- Abstract parent. Defines `family`, `collect_instances`, `create`,
  `update_instances`, `remove_instances`.
- Optional: `enabled`, `identifier`, `label`, `get_icon()`.
- Icon can be an image path, `qtawesome` name, `QPixmap` or `QIcon`.

### `Creator` (manual)
Triggered explicitly by artist via UI.
- `create_allow_context_change` — allow picking folder/task in create dialog
- `default_variant` / `default_variants` / `get_default_variant(s)` — pre-fill
- `description` / `detailed_description` (markdown) / `get_description`
- `pre_create_attr_defs` / `get_pre_create_attr_defs` — attrs shown in the
  create dialog
- `create(product_name, instance_data, pre_create_data)` — create the instance

### `HiddenCreator`
Not shown in UI; called by another creator. Signature:

```python
def create(self, instance_data, source_data): ...
```

### `AutoCreator`
Runs automatically on every `CreateContext.reset()` — use for instances the
artist shouldn't touch (e.g. `workfile`). `remove_instances()` is a no-op.

## Attribute definitions

Under `ayon_core.lib.attribute_definitions`:

- `BoolDef`, `NumberDef`, `TextDef`, `EnumDef`, `FileDef`, `UISeparatorDef`,
  `UILabelDef`, `HiddenDef`, `UILabelDef`.

Example on a Creator:

```python
from ayon_core.lib import attribute_definitions as ad

def get_instance_attr_defs(self):
    return [
        ad.BoolDef("render_farm", default=False, label="Render on Farm"),
        ad.NumberDef("priority", default=50, minimum=1, maximum=100),
    ]
```

These render in the Publisher's right-hand panel. The values live in
`instance["creator_attributes"]`.

## Dynamic product-name data

`get_dynamic_data(variant, task_name, asset_doc, project_name, host_name,
instance)` fills template keys. During create, `instance` is `None`:

```python
def get_dynamic_data(self, variant, task_name, asset_doc, project_name, host_name, instance):
    if instance is None:
        return {"layer": "{layer}"}
    return {"layer": instance.data["layer"]}
```

Studio product-name templates reference these keys (e.g.
`{family}{variant}{layer}`).

## Publish plugins

Standard pyblish plugin classes, with Ayon-aware mixins:

```python
import pyblish.api
from ayon_core.pipeline import OpenPypePyblishPluginMixin  # legacy name
from ayon_core.lib import attribute_definitions as ad


class MyCollector(pyblish.api.InstancePlugin, OpenPypePyblishPluginMixin):
    order = pyblish.api.CollectorOrder + 0.4
    label = "Collect My Thing"
    hosts = ["maya"]
    families = ["model"]

    optional = True  # Exposes toggle in the UI

    @classmethod
    def get_attribute_defs(cls):
        return [ad.BoolDef("process", default=cls.active, label=cls.label)]

    def process(self, instance):
        ...
```

- `hosts` — host name filter. `families` — product-type filter.
- Setting `optional = True` lets artists toggle the plugin per-instance; these
  overrides are persisted via `IPublishHost.update_context_data`.

`OpenPypePyblishPluginMixin.convert_attribute_values()` is the migration hook
for attribute schema changes (Ayon doesn't auto-purge old attr values).

## Validation errors

Three flavours, used to drive the error panel in the publisher:

### `PublishValidationError` — artist-fixable
```python
from ayon_core.pipeline import PublishValidationError

raise PublishValidationError(
    "Frame range mismatch",
    title="Frame Range",
    description="# Details\nExpected `1001-1100`, got `1000-1099`.",
    detail="Advanced: compare scene.startFrame vs folder.attrib.frameStart",
)
```

### `PublishXmlValidationError`
Like above but references `./help/<plugin_filename>.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<root>
  <error id="main">
    <title>Short Title</title>
    <description>## Markdown with `{formatting_keys}`</description>
    <detail>Advanced detail</detail>
  </error>
</root>
```

### `KnownPublishError`
Unrecoverable but expected (e.g. license server down). Full message is shown
to the artist.

## Loaders

Counterpart to Creators for pulling products **into** a scene. Under
`ayon_core.pipeline.load`:

```python
from ayon_core.pipeline.load import LoaderPlugin

class LoadAlembic(LoaderPlugin):
    families = ["model", "pointcache"]
    representations = ["abc"]
    label = "Load Alembic"
    icon = "file-import"
    order = -10

    def load(self, context, name, namespace, options): ...
    def update(self, container, representation): ...
    def remove(self, container): ...
```

## Scene inventory actions

Buttons/entries in the Scene Inventory tool. Subclass
`ayon_core.pipeline.InventoryAction` with `label`, `icon`, and a
`process(containers)` method.

## Registering products with the server

Handled by the **Integrator** plugin chain in `ayon-core` — you very rarely
implement this yourself. If you need to post-process right after integrate,
hook the `entity.version.created` / `entity.representation.created` events from
a service.

## Sources

- <https://docs.ayon.dev/docs/dev_publishing> — CCVEI, `CreatedInstance`, `CreateContext`, `BaseCreator`/`HiddenCreator`/`AutoCreator`/`Creator`, validation errors, attribute definitions, dynamic product names
- <https://docs.ayon.dev/docs/dev_addon_intro> — CCVEI + code snippets for `registered_host`, `CreateContext`, `get_containers`
- <https://docs.ayon.dev/docs/dev_colorspace> — `ColormanagedPyblishPluginMixin`, `set_representation_colorspace`, `colorspaceData` on representations
- <https://help.ayon.app/en/articles/1075843-creator-publisher> — artist-side explanation of Create vs Publish
- <https://help.ayon.app/en/articles/7070980-about-ayon-pipeline> — the four publishing principles (clear naming, versioning, immediate usability, immutability)
- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/pipeline/create> — creator source
- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/plugins/publish> — shipped publish plugins (reference implementations)
