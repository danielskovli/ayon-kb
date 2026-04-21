---
name: ayon-host-integration
description: Build or modify a DCC host integration for Ayon (new DCC integration, add host, wire DCC menu, bootstrap a DCC). Covers `HostBase`, `IWorkfileHost`, `ILoadHost`, `IPublishHost` / `INewPublisher`, `IHostAddon`, `install_host`, `register_host`, `registered_host`, `get_current_context`, `get_containers`, in-DCC bootstrap, menu wiring via `HostToolsHelper` / host tools, launch hooks, per-host folder layout. Use when adding a new DCC (e.g. `ayon-blender`, `ayon-maya`) or debugging host bootstrap / host tools menus.
---

# DCC host integration

A host is a client addon (see `ayon-addon-development`) plus an
implementation of `HostBase` + one or more host interfaces. `ayon-core`
provides the base classes; the host addon wires them to the DCC.

## Base + interfaces

```python
from ayon_core.host import HostBase, IWorkfileHost, ILoadHost
from ayon_core.pipeline import install_host


class MayaHost(HostBase, IWorkfileHost, ILoadHost):
    name = "maya"

    # --- IWorkfileHost (abstract methods only) ---
    def save_workfile(self, dst_path=None): ...
    def open_workfile(self, filepath): ...
    def get_current_workfile(self): ...

    # --- ILoadHost ---
    def get_containers(self): ...
```

Bootstrap (typically in the DCC's startup script):

```python
from ayon_core.pipeline import install_host
from ayon_my_dcc.api import MyDccHost

host = MyDccHost()
install_host(host)
```

After `install_host`, `registered_host()` returns the instance, the create/
load/publish/inventory plugin paths are active, and `get_current_context()`
resolves `{project_name, folder_path, task_name}`.

**One process handles one host at a time.** If you `install_host` again,
the previous one is replaced.

## Key interfaces

| Interface | Purpose | Representative methods |
|-----------|---------|-----------------------|
| `HostBase` | Base class | `name` attribute required |
| `IWorkfileHost` | Workfiles tool | abstract: `save_workfile`, `open_workfile`, `get_current_workfile`. Concrete helpers: `workfile_has_unsaved_changes`, `get_workfile_extensions`, `save_workfile_with_context`, `open_workfile_with_context`, `list_workfiles`, `list_published_workfiles`, `copy_workfile`, `copy_workfile_representation`. Deprecated shims (`save_file`, `open_file`, `current_file`, `has_unsaved_changes`, `file_extensions`) are still present — avoid in new code. |
| `ILoadHost` | Loader + Scene Inventory | `get_containers`, `update_context_data` |
| `IPublishHost` / `INewPublisher` | Publisher state | `get_context_data`, `update_context_data`, `get_context_title`, `get_current_context` |
| `IHostAddon` (on the `AYONAddon`, not the `HostBase`) | Registers host with `ayon-applications` | `get_workfile_extensions`, `add_implementation_envs` |

`IPublishHost` is critical — without it the publisher can't persist
per-scene plugin toggles. `get_context_data()` returns arbitrary JSON-able
data stored in the scene; `update_context_data(data, changes)` saves it.

Exact method sets drift between Ayon versions — inspect the current
`ayon-core/client/ayon_core/host/` source for the authoritative list.

## Recommended folder layout

```
ayon-<host>/
├─ package.py
├─ server/
└─ client/
   └─ ayon_<host>/
      ├─ __init__.py
      ├─ addon.py            # AYONAddon + IHostAddon
      ├─ version.py
      ├─ api/
      │   ├─ __init__.py     # {HostName}Host lives here
      │   ├─ pipeline.py
      │   └─ lib.py
      ├─ hooks/              # pre/post launch hooks
      ├─ startup/            # userSetup.py / menu.py / equivalent
      └─ plugins/
          ├─ create/
          ├─ load/
          ├─ publish/
          └─ inventory/
```

`ayon-core` discovers plugins automatically when the folder name under
`client/` matches the host `name`.

## In-DCC bootstrap pattern

Inside `api/pipeline.py`:

```python
from ayon_core.pipeline import (
    install_host, uninstall_host, registered_host, AVALON_CONTAINER_ID,
)
from ayon_core.pipeline.context_tools import (
    get_current_context, change_current_context,
)
```

After `install_host`, the bootstrap normally:

1. Registers scene-save/load callbacks for context tracking.
2. Creates the Ayon menu in the DCC's UI.
3. Loads context from env (`AYON_PROJECT_NAME`, `AYON_FOLDER_PATH`,
   `AYON_TASK_NAME`) set by the launcher.

## Menu wiring — `HostToolsHelper`

Unified tool launches across hosts:

```python
from ayon_core.tools.utils import host_tools

host_tools.show_workfiles()
host_tools.show_loader()
host_tools.show_publisher(tab="publish")    # or "create"
host_tools.show_scene_inventory()
host_tools.show_creator()
```

Typical DCC menu: **Workfiles · Create… · Load… · Publish… · Scene
Inventory · Library Loader · Set Frame Range**.

## Launch hooks

Live in `hooks/`. Common uses:

- Resolve the workfile to open; inject it into launch args.
- Set DCC plugin search paths (`MAYA_MODULE_PATH`, `NUKE_PATH`, …).
- Inject Python path for `ayon-core` and the host API.
- Copy / template-fill a workfile from a studio default.

See `ayon-addon-development` for the `PreLaunchHook` template.

## Publishing a new host — checklist

1. Scaffold the addon (see `ayon-addon-development`).
2. Subclass `HostBase + IWorkfileHost + ILoadHost + IPublishHost` in
   `api/`. Implement required methods.
3. Add `PreLaunchHook`s for env + Python path.
4. Bootstrap in `startup/` — `install_host(HostCls())`; wire the menu via
   `host_tools`.
5. Add plugins:
   - a `Creator` for `workfile` (start from `AutoCreator` in `ayon-core`)
   - a collector that collects the current workfile
   - one `LoaderPlugin` per format you want to support
6. Declare the app in `ayon-applications` settings (or extend it).
7. Build + upload; create a dev bundle; test via `ayon --use-dev`.

## Read live for reference

Cleanest references:
- `github.com/ynput/ayon-blender` — Python-first, minimal, clean
- `github.com/ynput/ayon-maya` — feature-complete, shows advanced patterns

## Deeper reading

- `07-host-integration.md` — full file with Sources
- Skill `ayon-publishing` — Creators / publish plugins
- Skill `ayon-loaders-inventory` — LoaderPlugin / InventoryAction

## Sources

- <https://docs.ayon.dev/docs/dev_host_implementation>
- <https://docs.ayon.dev/docs/dev_publishing> — `IPublishHost`

### Source repos

- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/host> — `HostBase` + interfaces
- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/pipeline> — `install_host`, `registered_host`, `AVALON_CONTAINER_ID`
- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/tools/utils> — `HostToolsHelper`, `host_tools.show_*`
- <https://github.com/ynput/ayon-applications> — `PreLaunchHook`, `Application`, `ApplicationManager`
- <https://github.com/ynput/ayon-blender> — clean Python-only reference integration
- <https://github.com/ynput/ayon-maya> — feature-complete reference
- <https://github.com/ynput/ayon-houdini> — USD-heavy reference
- <https://github.com/ynput/ayon-nuke> — DCC-extension (menu.py) pattern reference
