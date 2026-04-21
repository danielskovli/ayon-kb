---
name: ayon-loaders-inventory
description: Write Ayon loader plugins and scene inventory actions — `LoaderPlugin` class attributes and methods, `ProductLoaderPlugin`, `LoaderHookPlugin`, utility functions (`load_container`, `update_container`, `switch_container`, `remove_container`, `get_representation_path*`, `get_representation_context*`), container dict schema, custom `InventoryAction`. Use when writing a loader for a new format, adding Scene Inventory actions, resolving published file paths outside a DCC, or debugging "Update to Latest" / "Switch Asset".
when_to_use: Triggered by "loader", "LoaderPlugin", "ProductLoaderPlugin", "LoaderHookPlugin", "load_container", "switch_container", "update_container", "InventoryAction", "scene inventory", "get_containers", "get_representation_path", "container dict", "AVALON_CONTAINER_ID", "hero version".
---

# Ayon loaders & scene inventory

Loaders pull a **published representation** into the current scene. They
live under `client/<addon>/plugins/load/` and are discovered via
`IPluginPaths.get_plugin_paths()["load"]`.

## `LoaderPlugin`

From `ayon_core.pipeline.load.plugins`:

```python
from ayon_core.pipeline import LoaderPlugin


class LoadAlembic(LoaderPlugin):
    product_types   = {"pointcache", "model", "animation"}
    representations = {"abc"}
    extensions      = {"abc"}       # default {"*"}
    order           = 0
    enabled         = True

    label = "Load Alembic"
    icon  = "file-import"
    color = "orange"

    is_multiple_contexts_compatible = False

    def load(self, context, name, namespace, options):
        """context = {project, folder, task, product, version,
                       representation, file_path}. Must return a container dict."""

    def update(self, container, context):
        """Switch a container to a different representation."""

    def remove(self, container):
        """Remove and return bool."""

    @classmethod
    def get_options(cls, contexts):
        return []      # optional attribute defs for a 'Load options' dialog
```

### Class attributes

| Attr | Use |
|------|-----|
| `skip_discovery` | `True` on bases; subclasses omit → discovered |
| `product_types` | Compatible product types (set[str]) |
| `product_base_types` | Optional base-type filter |
| `representations` | Compatible representation names |
| `extensions` | Supported extensions (default `{"*"}`) |
| `order` | Ordering among matching loaders |
| `is_multiple_contexts_compatible` | Accept multi-selection loads |
| `enabled` | Disable without removing |
| `label`, `icon`, `color` | UI |

### Helper methods

- `apply_settings(project_settings)` — inject settings values
- `is_compatible_loader(context)` — default compatibility (product type +
  representation + extension); override to customise
- `has_valid_extension(repre_entity)` — extension match
- `get_representations()` — compatible names
- `get_representation_name_aliases(name)` — fallback names when updating
  (for server-side renames)

## Specialised bases

- **`ProductLoaderPlugin`** — loads at the **product** level, resolves its
  own latest version internally. Use when the loader operates on "the
  product" rather than a specific representation.
- **`LoaderHookPlugin`** — non-invasive wrapper. Declares `pre_load` /
  `post_load` / `pre_update` / `post_update` / `pre_remove` / `post_remove`
  and `is_compatible(Loader)`. Use for cross-cutting concerns (telemetry,
  auto-apply overrides).

## Utility functions

From `ayon_core.pipeline.load.utils`:

```python
load_container(Loader, representation, namespace=None, name=None,
               options=None, **kwargs)
load_with_repre_context(Loader, repre_context, ...)
load_with_product_context(Loader, product_context, ...)
load_with_product_contexts(Loader, product_contexts, ...)

remove_container(container)
update_container(container, version=-1)              # -1 = latest
switch_container(container, representation, loader_plugin=None)
```

### Context resolution

```python
get_representation_context(project_name, representation)
get_representation_contexts(project_name, repre_entities)
get_repres_contexts(repre_ids, project_name=None)    # legacy
get_product_contexts(product_ids, project_name=None)

get_representation_by_names(project_name, folder_path,
                            product_name, version_name, representation_name)
```

### Path resolution (uses Anatomy)

```python
from ayon_core.pipeline.anatomy import Anatomy
from ayon_core.pipeline.load import (
    get_representation_path, get_representation_path_by_names,
    get_representation_path_from_context,
)
```

### Introspection

```python
discover_loader_plugins()
get_loaders_by_name()
is_compatible_loader(Loader, context)
get_loader_identifier(loader)
```

## Container dict

Every `load()` must return a **container** describing what was loaded.
Shape varies per host but typically:

| Key | Meaning |
|-----|---------|
| `schema` | `"openpype:container-2.0"` (legacy name kept) |
| `id` | `AVALON_CONTAINER_ID` constant |
| `name` | User-friendly name for the inventory |
| `namespace` | Host-specific scope for the loaded objects |
| `loader` | Loader class name (so updates/switches re-pick correctly) |
| `representation` | Representation id |
| `objectName` | Host node/object that groups the load |

`host.get_containers()` (from `ILoadHost`) returns the list.

## Scene Inventory UI — what artists see

The tool is at `AYON → Manage → Inventory`. Its actions map to:

| Action | Under the hood |
|--------|---------------|
| Update to Latest | `update_container(container, -1)` |
| Change to Hero | `update_container(container, HeroVersionType())` |
| Set Version | `update_container(container, specific_version)` |
| Switch Asset | `switch_container(container, new_representation)` |
| Remove | `remove_container(container)` |
| Download/Upload | enqueue a Site Sync transfer |

Outdated containers go red. Hero versions show `[v003]` brackets for their
source version.

## Custom inventory actions

```python
from ayon_core.pipeline import InventoryAction


class FreezeTransforms(InventoryAction):
    label = "Freeze Transforms"
    icon  = "snowflake"
    color = "white"
    order = 10

    def is_compatible(self, container):
        return container.get("loader") in {"LoadAlembic", "LoadMaScene"}

    def process(self, containers):
        ...
        return True          # triggers an inventory refresh
```

Register via `IPluginPaths.get_plugin_paths()["inventory"]`.

## Downloading outside a DCC

```python
import ayon_api
from ayon_core.pipeline.load import (
    get_representation_by_names, get_representation_path,
)
from ayon_core.pipeline.anatomy import Anatomy

project = "my_project"
repre = get_representation_by_names(
    project, "/seq010/sh010", "modelMain", "v003", "abc",
)
path = get_representation_path(project, repre, anatomy=Anatomy(project))
```

For Site-Sync-aware downloads, call the site-sync addon's API (see
`ayon-site-sync-colorspace`) — the local path may not yet hold the file.

## Tips / traps

- Keep `load()` idempotent enough to re-run (DCCs do cancel/retry).
- Namespace everything — collisions are the #1 source of "my load broke."
- `switch_container` is for changing product/variant/representation, not
  just version.
- Post-import steps (apply a look, freeze transforms, …) belong in a
  `LoaderHookPlugin` — don't hard-code them into `load()`.
- For `update()`, compare `container["representation"]` to the new
  representation id; no-op when equal.

## Deeper reading

- `15-loaders-inventory.md` — full file with Sources
- Skill `ayon-publishing` — the symmetric "push side"
- Skill `ayon-anatomy-templates` — how `get_representation_path` resolves

## Sources

- <https://help.ayon.app/en/articles/4345209-loader>
- <https://help.ayon.app/en/articles/9770233-scene-inventory>
- <https://docs.ayon.dev/docs/dev_addon_intro>

### Source repos

- <https://github.com/ynput/ayon-core/blob/develop/client/ayon_core/pipeline/load/plugins.py> — `LoaderPlugin`, `ProductLoaderPlugin`, `LoaderHookPlugin` source
- <https://github.com/ynput/ayon-core/blob/develop/client/ayon_core/pipeline/load/utils.py> — `load_container`, `update_container`, `switch_container`, `remove_container`, `get_representation_path*`, `discover_loader_plugins`, `get_loaders_by_name`
- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/pipeline> — `InventoryAction`, `AVALON_CONTAINER_ID`
- <https://github.com/ynput/ayon-maya/tree/develop/client/ayon_maya/plugins/load> — reference loader implementations
- <https://github.com/ynput/ayon-maya/tree/develop/client/ayon_maya/plugins/inventory> — reference `InventoryAction` implementations (e.g. `ConnectGeometry`)
- <https://github.com/ynput/ayon-houdini/tree/develop/client/ayon_houdini/plugins/load> — HDA-backed loaders
