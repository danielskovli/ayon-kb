# 07 — DCC Host integration

A **host** is an Ayon integration for a specific DCC (Maya, Nuke, Blender, …).
It is a normal client addon (see `06-client-addon.md`) plus an implementation of
`HostBase` + one or more host interfaces.

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

Install from inside the DCC's startup (once, in-process):

```python
from ayon_core.pipeline import install_host
from ayon_my_dcc.api import MyDccHost

host = MyDccHost()
install_host(host)
```

After `install_host`:
- `ayon_core.pipeline.registered_host()` returns the instance.
- Global plugin registration runs (create/load/publish paths get picked up).
- **One process can only host one DCC at a time.**

## Required / useful interfaces

| Interface | Methods | Purpose |
|-----------|---------|---------|
| `HostBase` | `name` attribute is required | base class |
| `IWorkfileHost` | abstract: `save_workfile`, `open_workfile`, `get_current_workfile`. concrete helpers: `workfile_has_unsaved_changes`, `get_workfile_extensions`, `save_workfile_with_context`, `open_workfile_with_context`, `list_workfiles`, `list_published_workfiles`, `copy_workfile`, `copy_workfile_representation`. Deprecated shims (`save_file`, `open_file`, `current_file`, `has_unsaved_changes`, `file_extensions`) are still present for back-compat — don't use in new code. | Workfiles tool |
| `ILoadHost` | `get_containers`, `update_context_data` (varies) | Loader / scene inventory |
| `IPublishHost` / `INewPublisher` | `get_context_data`, `update_context_data`, `get_context_title`, `get_current_context` | Publisher (stores pyblish plugin states) |
| `IHostAddon` (on the `AYONAddon`) | `get_workfile_extensions`, `add_implementation_envs` | Registers the host with `ayon-applications` |

`IPublishHost` is critical — without it the new publisher can't persist per-
scene plugin toggles. `get_context_data()` returns arbitrary JSON-able data
stored in the scene; `update_context_data(data, changes)` saves it back.

## Folder layout

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
      │   ├─ __init__.py     # defines {HostName}Host
      │   ├─ pipeline.py
      │   ├─ lib.py
      │   └─ ...
      ├─ hooks/              # pre/post launch hooks
      ├─ startup/            # scripts the DCC runs on boot (userSetup.py, etc.)
      └─ plugins/
          ├─ create/
          ├─ load/
          ├─ publish/
          └─ inventory/
```

Folder name under `client/` must match the host `name` for autodiscovery of
hooks & plugins to work cleanly.

## Host UI integration

`HostToolsHelper` (from `ayon_core.tools.utils.host_tools`) unifies tool launches
across hosts. Wire the DCC menu to it:

```python
from ayon_core.tools.utils import host_tools

host_tools.show_workfiles()
host_tools.show_loader()
host_tools.show_publisher(tab="publish")
host_tools.show_scene_inventory()
host_tools.show_creator()
```

What you usually add to the DCC menu bar:

- Workfiles
- Create…
- Load…
- Publish…
- Scene Inventory
- Library Loader (optional)
- Set Frame Range (optional)

## Launch hooks

Live in `hooks/`. Typical uses:

- Resolve the workfile to open and inject it into launch args.
- Set DCC plugin search paths (`MAYA_MODULE_PATH`, `NUKE_PATH`, …).
- Inject Python path for `ayon-core` and the host API.
- Copy/template workfile from a studio default.

See `06-client-addon.md#launch-hooks` for the `PreLaunchHook` pattern.

## In-DCC bootstrap

After `install_host`, bootstrap typically:

1. Registers a scene-save/load callback for context tracking.
2. Creates the Ayon menu via the DCC's UI API.
3. Loads workfile context from env (`AYON_PROJECT_NAME`, `AYON_FOLDER_PATH`,
   `AYON_TASK_NAME`).

Common imports inside `api/pipeline.py`:

```python
from ayon_core.pipeline import (
    install_host, uninstall_host, registered_host,
    AVALON_CONTAINER_ID,
)
from ayon_core.pipeline.context_tools import (
    get_current_context, change_current_context,
)
```

## Read live for reference

The cleanest integrations to read are `ayon-blender` (Python-first DCC, simple)
and `ayon-maya` (feature-complete). Both are at `github.com/ynput/ayon-<host>`.

## Sources

- <https://docs.ayon.dev/docs/dev_host_implementation> — `HostBase`, `IWorkfileHost`, `ILoadHost`, `INewPublisher`, `install_host`, hooks
- <https://docs.ayon.dev/docs/dev_publishing> — `IPublishHost` (`get_context_data`, `update_context_data`, `get_context_title`, `get_current_context`)
- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/host> — `HostBase` + interface source
- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/tools/utils> — `HostToolsHelper`, `host_tools.show_*`
- <https://github.com/ynput/ayon-blender> — clean Python-only reference integration
- <https://github.com/ynput/ayon-maya> — full-featured reference integration
