# 06 — Client-side addon

The **client half** lives under `client/<name>/` and ships to artist machines
via the launcher. It runs under the launcher's Python, and (after host install)
also inside DCC processes.

## Minimal `client/my_addon/__init__.py`

```python
from ayon_core.addon import AYONAddon
from .version import __version__


class MyAddon(AYONAddon):
    label = "My Addon"
    name = "my_addon"
    version = __version__

    def initialize(self, settings):
        """Called with merged (studio+project) settings for this addon."""
        pass
```

Notes:
- `name` **must match** `package.py`'s `name`.
- `initialize(settings)` receives the resolved settings dict for this bundle.
- Heavy imports (Qt, DCC-specific) should stay lazy — `ayon-core` imports
  every addon on launcher boot just to discover them.

## Common interfaces

`ayon-core` exposes mixin interfaces the addon opts into:

| Interface | Adds |
|-----------|------|
| `ITrayAddon` | Hooks into the launcher's system-tray UI |
| `ITrayAction` / `ITrayService` | Simpler tray helpers |
| `IPluginPaths` | Registers extra plugin paths (create/load/publish/inventory) |
| `IHostAddon` | Marks the addon as providing a DCC host |
| `IWorkfileHost` | Implements workfile ops (see `07-host-integration.md`) |
| `ILoadHost` | Implements load/inventory ops |
| `IPublishHost` / `INewPublisher` | Implements publish context storage |

### `ITrayAddon`

```python
from ayon_core.addon import AYONAddon, ITrayAddon
from qtpy import QtWidgets


class MyAddon(AYONAddon, ITrayAddon):
    name = "my_addon"
    label = "My Addon"

    def tray_init(self): ...
    def tray_start(self): ...
    def tray_exit(self): ...

    def tray_menu(self, tray_menu):
        menu = QtWidgets.QMenu(self.label, tray_menu)
        action = QtWidgets.QAction("Show Dialog", menu)
        action.triggered.connect(self.show_dialog)
        menu.addAction(action)
        tray_menu.addMenu(menu)

    def show_dialog(self):
        QtWidgets.QMessageBox.information(None, "Hi", "Hello AYON!")
```

### `IPluginPaths`

The standard way to ship publish/create/load/inventory plugins for one or more
hosts:

```python
def get_plugin_paths(self):
    return {
        "create":  ["/abs/path/to/create"],
        "load":    ["/abs/path/to/load"],
        "publish": ["/abs/path/to/publish"],
        "actions": ["/abs/path/to/actions"],
    }
```

Per-host filtering is typically done by plugin class attributes (`hosts =
["maya"]`).

You can also return nested dicts keyed by host name to scope plugins explicitly.

## CLI — `click_wrap`

`ayon-core` wraps `click` with `click_wrap` so addons can contribute CLI commands
that run under the launcher's Python environment. Used heavily by web actions
and by developer tooling.

```python
from ayon_core.addon import AYONAddon, click_wrap


class MyAddon(AYONAddon):
    name = "my_addon"

    def cli(self, click_group):
        click_group.add_command(cli_main.to_click_obj())


@click_wrap.group(MyAddon.name, help="My Addon commands")
def cli_main():
    pass


@cli_main.command()
@click_wrap.option("--project", type=str, required=False)
@click_wrap.option("--entity-id", type=str, required=False)
def show_selected_path(project, entity_id):
    from qtpy import QtWidgets
    QtWidgets.QMessageBox.information(
        None, "Action Triggered",
        f"{project}/{entity_id}",
    )
```

Invoke from launcher:

```bash
ayon addon my_addon show-selected-path --project foo --entity-id abc
```

(or `ayon_console.bat --use-dev addon ...` during development)

## Settings on the client

The `initialize(settings)` arg is a dict with the addon's merged settings
(values already reflect studio + project + site overrides). Store what you
need on `self`; don't re-query the server each operation.

```python
def initialize(self, settings):
    self.enabled = settings.get("enabled", True)
    self.studio_name = settings.get("studio_name", "")
```

## Launch hooks

Hooks live in `client/<name>/hooks/` (auto-discovered when folder name matches
the host name convention) or register explicitly. Two kinds:

- **Pre-launch hooks** — modify subprocess env / args before DCC starts.
- **Post-launch hooks** — run after subprocess has started.

Hooks in the same launch share a `LaunchContext` for passing data.

Typical signature:

```python
from ayon_applications import PreLaunchHook

class SetRenderEnv(PreLaunchHook):
    # one or both of these narrows applicability
    app_groups = {"maya"}
    hosts = {"maya"}
    order = 10

    def execute(self):
        self.launch_context.env["MY_FLAG"] = "1"
```

The `ayon-applications` addon owns the application definitions and launch
pipeline.

## Entry point summary

| File | Purpose |
|------|---------|
| `client/<name>/__init__.py` | Re-export the addon class |
| `client/<name>/addon.py` | `AYONAddon` subclass |
| `client/<name>/version.py` | `__version__ = "..."` |
| `client/<name>/plugins/...` | Create/Load/Publish/Inventory plugins |
| `client/<name>/hooks/...` | Pre/post launch hooks |
| `client/<name>/api/` | In-DCC code (imports DCC modules; keep lazy) |
| `client/<name>/startup/` | Scripts run by the DCC on open (`userSetup.py` etc.) |
| `client/pyproject.toml` | Declares Python deps that go in the bundle's dep package |

## Tips

- Name everything `snake_case`. Class names stay PascalCase.
- Every import of a DCC module (`import maya.cmds`) must be inside a function
  body, not at module top level, or the launcher will fail on machines without
  that DCC.
- Use `ayon_core.lib.Logger.get_logger(__name__)` for logging.
- `ayon_core.pipeline.registered_host()` gets the currently installed host
  inside DCC context.
- `ayon_core.pipeline.context_tools.get_current_context()` returns
  `{project_name, folder_path, task_name}`.

## Sources

- <https://docs.ayon.dev/docs/dev_addon_creation> — `AYONAddon`, tray integration, `IPluginPaths`, CLI via `click_wrap`
- <https://docs.ayon.dev/docs/dev_host_implementation> — launch hooks pattern
- <https://docs.ayon.dev/docs/dev_addon_intro> — `registered_host`, `get_current_context`, `CreateContext` imports
- <https://docs.ayon.dev/ayon-core/> — `ayon-core` generated reference
- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/addon> — `AYONAddon`, interface mixins source
