# 15 — Loaders & Scene Inventory

## Loader plugins

A `LoaderPlugin` pulls a **published representation** into the current scene.
They live under `client/<addon>/plugins/load/` and are discovered via
`IPluginPaths.get_plugin_paths()["load"]`.

### Base class

From `ayon_core.pipeline.load.plugins`:

```python
from ayon_core.pipeline import LoaderPlugin


class LoadAlembic(LoaderPlugin):
    product_types   = {"pointcache", "model", "animation"}
    representations = {"abc"}
    extensions      = {"abc"}            # default {"*"}
    order           = 0                  # plugin ordering
    enabled         = True

    label     = "Load Alembic"
    icon      = "file-import"
    color     = "orange"

    is_multiple_contexts_compatible = False

    def load(self, context, name, namespace, options):
        """Load one representation into the scene.

        context = {
            "project": {...}, "folder": {...}, "task": {...},
            "product": {...}, "version": {...}, "representation": {...},
            "file_path": "/abs/path/to/file.abc",
        }
        Must return a container dict describing what was loaded.
        """

    def update(self, container, context):
        """Switch a loaded container to a different representation."""

    def remove(self, container):
        """Remove the container from the scene. Return bool."""

    @classmethod
    def get_options(cls, contexts):
        """Optional attribute-defs shown in a 'Load options' dialog."""
        return []
```

### Key class attributes (from `ayon-core` source)

| Attribute | Purpose |
|-----------|---------|
| `skip_discovery` | `True` on base classes; subclasses omit → discovered |
| `product_types: set[str]` | Compatible product types |
| `product_base_types: Optional[set[str]]` | Optional base-type filter |
| `representations: set[str]` | Compatible representation names |
| `extensions: set[str]` | Supported file extensions (default `{"*"}`) |
| `order: int` | Ordering among matching loaders |
| `is_multiple_contexts_compatible: bool` | Accept multi-select loads |
| `enabled: bool` | Disable without removing |
| `label`, `icon`, `color` | UI |

### Key helper methods

- `apply_settings(project_settings)` — hook called by the plugin system to
  inject settings values.
- `is_compatible_loader(context)` — default compatibility check
  (product type + representation + extension). Override to customise.
- `has_valid_extension(repre_entity)` — extension match.
- `get_representations()` — returns the compatible names.
- `get_representation_name_aliases(representation_name)` — fallback names
  when updating (e.g. representation was renamed server-side).

### Specialised base classes

- **`ProductLoaderPlugin`** — same family of methods but loads at the
  **product** level, iterating its own latest version. Used when a loader
  operates on "the product" rather than a specific representation
  (e.g. some Houdini loaders).
- **`LoaderHookPlugin`** — non-invasive wrapper. Declares `pre_load` /
  `post_load` / `pre_update` / `post_update` / `pre_remove` / `post_remove`.
  Compatibility via `is_compatible(Loader)`. Use for cross-cutting
  concerns (telemetry, extra tagging, auto-apply of overrides).

## Utility functions

From `ayon_core.pipeline.load.utils`:

```python
load_container(Loader, representation, namespace=None,
               name=None, options=None, **kwargs)
load_with_repre_context(Loader, repre_context, ...)
load_with_product_context(Loader, product_context, ...)
load_with_product_contexts(Loader, product_contexts, ...)

remove_container(container)
update_container(container, version=-1)          # -1 = latest
switch_container(container, representation, loader_plugin=None)
```

Context helpers (use these instead of custom GraphQL queries when you just
need the full parent chain):

```python
get_representation_context(project_name, representation)
get_representation_contexts(project_name, representation_entities)
get_repres_contexts(representation_ids, project_name=None)      # legacy
get_product_contexts(product_ids, project_name=None)

get_representation_by_names(project_name, folder_path,
                            product_name, version_name,
                            representation_name)
```

Path resolution (uses Anatomy — see `14-anatomy-templates.md`):

```python
get_representation_path(project_name, repre_entity,
                        *, anatomy=None, project_entity=None)
get_representation_path_by_names(project_name, folder_path,
                                 product_name, version_name,
                                 representation_name,
                                 anatomy=None)
get_representation_path_from_context(context)
```

Introspection:

```python
discover_loader_plugins()      # returns all registered loaders
get_loaders_by_name()           # dict of name -> plugin
is_compatible_loader(Loader, context)
get_loader_identifier(loader)
```

## Container dict

Every `load()` must return a **container** describing what was loaded. Shape
is host-defined but typically includes:

| Key | Meaning |
|-----|---------|
| `schema` | Usually `"openpype:container-2.0"` (legacy name still used) |
| `id` | `AVALON_CONTAINER_ID` constant — marks this as a container |
| `name` | User-friendly name in the inventory |
| `namespace` | Host-specific scope for the loaded objects |
| `loader` | Loader class name (so updates/switches pick the right plugin) |
| `representation` | Representation id |
| `objectName` | DCC-specific node/object that groups the load |

The Scene Inventory reads these dicts by calling
`host.get_containers()` (implemented by `ILoadHost`).

## Scene Inventory UI

The artist-facing tool (see `17-user-workflows.md#scene-inventory`) calls:

- **Update to Latest** → `update_container(container, -1)` for each
  outdated container.
- **Change to Hero** → `update_container(container, "hero")`.
- **Set version** → `update_container(container, specific_version)`.
- **Switch Asset** → `switch_container(container, new_representation)`.
- **Remove** → `remove_container(container)`.
- **Download/Upload** → queues a site-sync transfer on the representation.

## Inventory actions (custom)

Want a new button in Scene Inventory? Subclass `InventoryAction`:

```python
from ayon_core.pipeline import InventoryAction


class FreezeTransforms(InventoryAction):
    label  = "Freeze Transforms"
    icon   = "snowflake"
    color  = "white"
    order  = 10

    def is_compatible(self, container):
        return container.get("loader") in {"LoadAlembic", "LoadMaScene"}

    def process(self, containers):
        """Containers = list of container dicts the user selected."""
        ...
        return True      # True triggers an inventory refresh
```

Register by returning its path from `IPluginPaths.get_plugin_paths()["inventory"]`.

## Tips

- Keep `load()` idempotent enough to re-run (DCCs do cancel/retry).
- Namespace everything — name collisions are the single biggest source of
  "my load broke" bugs.
- If your format needs post-processing (e.g. apply a look after import), that
  belongs in a `LoaderHookPlugin`, **not** hard-coded into the loader.
- For `update()`, compare `container["representation"]` to
  `context["representation"]["id"]` — if equal, no-op.
- `switch_container` is used when changing product/variant/representation —
  not just version. The Scene Inventory's "Switch Asset" action calls this.

## Sources

- <https://github.com/ynput/ayon-core/blob/develop/client/ayon_core/pipeline/load/plugins.py> — `LoaderPlugin`, `ProductLoaderPlugin`, `LoaderHookPlugin` source
- <https://github.com/ynput/ayon-core/blob/develop/client/ayon_core/pipeline/load/utils.py> — `load_container`, `remove_container`, `switch_container`, `get_representation_path*` source
- <https://help.ayon.app/en/articles/4345209-loader> — artist-facing Loader UI (search, filters, hero brackets `[v003]`, grouping, site widget)
- <https://help.ayon.app/en/articles/9770233-scene-inventory> — artist-facing Scene Inventory actions (Update, Change to Hero, Switch Asset, Remove, Download/Upload)
- <https://docs.ayon.dev/docs/dev_addon_intro> — `host.get_containers()` example
