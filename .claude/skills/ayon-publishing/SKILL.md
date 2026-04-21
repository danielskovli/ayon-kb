---
name: ayon-publishing
description: Write Ayon publish plugins — pyblish CCVEI lifecycle (Collector, Validator, Extractor, Integrator) plus the Create phase. Use when writing publishers / creators / validators / extractors / integrators, debugging publish failures, defining a new productType / family / product type, or adding a repair action. Covers `CreateContext`, `CreatedInstance`, `instance_id`. Creator flavours: `BaseCreator`, `Creator`, `HiddenCreator`, `AutoCreator`. Attribute definitions (`attribute_definitions`, `pre_create_attr_defs`): `BoolDef`, `NumberDef`, `EnumDef`, `TextDef`, `UISeparatorDef`. Dynamic product names via `get_dynamic_data`. Validation error classes: `PublishValidationError`, `PublishXmlValidationError`, `KnownPublishError`.
---

# Ayon publishing

Publishing is **pyblish** with a layer on top that adds typed Creators,
artist-friendly error reporting, and automatic registration with the Ayon
server.

## CCVEI lifecycle

| Stage | Pyblish order | Purpose |
|-------|---------------|---------|
| **C**ollect | 0–0.49 | Scan scene, gather metadata |
| **V**alidate | 0.5–0.99 | Integrity checks — fail early, artist-readable |
| **E**xtract | 1.0–1.99 | Write output files to staging |
| **I**ntegrate | 2.0+ | Upload + register on server |

Constants: `pyblish.api.CollectorOrder`, `ValidatorOrder`, `ExtractorOrder`,
`IntegratorOrder`.

The **Create** phase sits before CCVEI: artists mark what will be
published. Full lifecycle: **Create → Collect → Validate → Extract →
Integrate**.

## `CreatedInstance`

Dict-like, lives in scene metadata. Common keys:

| Key | Notes |
|-----|------|
| `id` | const `"pyblish.avalon.instance"` |
| `instance_id` | UUID |
| `family` / `productType` | e.g. `model`, `render` |
| `creator_identifier` | Which Creator owns this |
| `creator_attributes` | Creator-defined dict |
| `variant` | `Main`, `High`, … |
| `productName` | Resolved via template |
| `active` | Toggle in UI |
| `folderPath` / `asset`, `task` | Target context |

`CreatedInstance.data_to_store()` serialises for scene persistence.

## `CreateContext`

Controller for the Create phase (`ayon_core.pipeline.create`):

```python
from ayon_core.pipeline import registered_host
from ayon_core.pipeline.create import CreateContext

host = registered_host()
ctx = CreateContext(host)
for inst in ctx.instances:
    print(inst.data["productName"])
```

It discovers creators + publish plugins, calls `collect_instances()` on
every creator, registers instances by id, and validates that the host
implements `IPublishHost`.

## Creator flavours

All under `ayon_core.pipeline.create`:

### `BaseCreator`
Abstract parent. Defines `family`, `collect_instances`, `create`,
`update_instances`, `remove_instances`. Optional: `enabled`, `identifier`,
`label`, `get_icon()`.

### `Creator` — manual
Triggered by artist via UI.

- `create_allow_context_change` — allow folder/task change at create time
- `default_variant` / `default_variants` (or `get_default_variant(s)`)
- `description` / `detailed_description` (markdown)
- `pre_create_attr_defs` — attrs shown in the create dialog
- `create(product_name, instance_data, pre_create_data)` — emits instance

### `HiddenCreator`
Not shown in UI; called by another creator.

```python
def create(self, instance_data, source_data): ...
```

### `AutoCreator`
Runs automatically on `CreateContext.reset()`. Use for instances the artist
shouldn't touch (e.g. `workfile`). `remove_instances()` is a no-op.

## Attribute definitions

From `ayon_core.lib.attribute_definitions`:

`BoolDef`, `NumberDef`, `TextDef`, `EnumDef`, `FileDef`, `UISeparatorDef`,
`UILabelDef`, `HiddenDef`.

```python
from ayon_core.lib import attribute_definitions as ad

def get_instance_attr_defs(self):
    return [
        ad.BoolDef("render_farm", default=False, label="Render on Farm"),
        ad.NumberDef("priority", default=50, minimum=1, maximum=100),
    ]
```

Values live in `instance["creator_attributes"]`.

## Dynamic product names

`get_dynamic_data(variant, task_name, asset_doc, project_name, host_name,
instance)` fills template keys. During create, `instance` is `None`:

```python
def get_dynamic_data(self, variant, task_name, asset_doc,
                    project_name, host_name, instance):
    if instance is None:
        return {"layer": "{layer}"}
    return {"layer": instance.data["layer"]}
```

Studio product-name templates reference these keys (e.g.
`{family}{variant}{layer}`).

## Publish plugins

Standard pyblish classes with the `OpenPypePyblishPluginMixin` (current
class name — not a bug):

```python
import pyblish.api
from ayon_core.pipeline import OpenPypePyblishPluginMixin
from ayon_core.lib import attribute_definitions as ad


class MyCollector(pyblish.api.InstancePlugin, OpenPypePyblishPluginMixin):
    order    = pyblish.api.CollectorOrder + 0.4
    label    = "Collect My Thing"
    hosts    = ["maya"]
    families = ["model"]
    optional = True

    @classmethod
    def get_attribute_defs(cls):
        return [ad.BoolDef("process", default=cls.active, label=cls.label)]

    def process(self, instance):
        ...
```

- `hosts` — host filter. `families` — product-type filter.
- `optional = True` → per-instance toggle in the UI (persisted via
  `IPublishHost.update_context_data`).
- Override `convert_attribute_values()` to migrate legacy attr values
  (Ayon doesn't auto-purge old ones).

## Validation errors

```python
from ayon_core.pipeline import (
    PublishValidationError, PublishXmlValidationError, KnownPublishError,
)
```

### `PublishValidationError` — artist-fixable

```python
raise PublishValidationError(
    "Frame range mismatch",
    title="Frame Range",
    description="# Details\nExpected `1001-1100`, got `1000-1099`.",
    detail="Advanced: compare scene.startFrame vs folder.attrib.frameStart",
)
```

### `PublishXmlValidationError`

References `./help/<plugin_filename>.xml`:

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

### `KnownPublishError` — unrecoverable but expected
Full message shown to the artist (e.g. license server down).

## Repair actions (self-healing validators)

Attach pyblish `Action`s to a plugin for one-click fixes:

```python
import pyblish.api

class RepairPivot(pyblish.api.Action):
    label = "Reset pivot to origin"
    on    = "failed"
    icon  = "wrench"

    def process(self, context, plugin):
        for instance in context:
            ...   # mutate the scene

class ValidatePivot(pyblish.api.InstancePlugin):
    order    = pyblish.api.ValidatorOrder
    hosts    = ["maya"]
    families = ["model"]
    actions  = [RepairPivot]

    def process(self, instance):
        if instance.data.get("pivot_off_origin"):
            raise PublishValidationError(
                "Pivot is not at origin", title="Pivot",
                description="Use *Repair* or move pivot manually.",
            )
```

## Registering products

The **Integrator** plugin chain in `ayon-core` uploads files + calls the
REST API (via `ayon-python-api`) — you rarely implement this yourself. For
post-integration reactions, subscribe to `entity.version.created` or
`entity.representation.created` from a service (see
`ayon-events-services`).

## Copy-as-scaffold pattern

Fastest way to ship a new product type: copy a shipped plugin from
`ayon-core/client/ayon_core/plugins/publish/` and specialise.

## Deeper reading

- `08-publishing.md` — full file with Sources
- Skill `ayon-loaders-inventory` — the symmetric "pull side"
- Skill `ayon-host-integration` — `IPublishHost`

## Sources

- <https://docs.ayon.dev/docs/dev_publishing>
- <https://docs.ayon.dev/docs/dev_addon_intro>
- <https://docs.ayon.dev/docs/dev_colorspace>
- <https://help.ayon.app/en/articles/1075843-creator-publisher>

### Source repos

- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/pipeline/create> — `CreateContext`, `CreatedInstance`, `BaseCreator`, `Creator`, `HiddenCreator`, `AutoCreator`
- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/pipeline/publish> — `PublishValidationError`, `PublishXmlValidationError`, `KnownPublishError`, `OpenPypePyblishPluginMixin`
- <https://github.com/ynput/ayon-core/blob/develop/client/ayon_core/lib/attribute_definitions.py> — `BoolDef`, `NumberDef`, `EnumDef`, etc.
- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/plugins/publish> — reference shipped publish plugins (copy-as-scaffold)
- <https://github.com/ynput/ayon-core/blob/develop/client/ayon_core/pipeline/colorspace.py> — `ColormanagedPyblishPluginMixin`
