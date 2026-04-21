---
name: ayon-addon-development
description: Build Ayon addons — repo layout, `package.py`, server side (`BaseServerAddon`, `BaseSettingsModel`, `SettingsField`, REST endpoints, web actions, frontend scopes, static public/private files) and client side (`AYONAddon`, `ITrayAddon`, `IPluginPaths`, `click_wrap` CLI, launch hooks). Use when creating, extending, or debugging an Ayon addon; defining settings; wiring server endpoints; writing tray widgets; adding frontend UIs; or packaging/uploading an addon.
when_to_use: Triggered by "build an addon", "new Ayon addon", "BaseServerAddon", "AYONAddon", "package.py", "addon settings", "SettingsField", "add_endpoint", "web action", "frontend_scopes", "IPluginPaths", "ITrayAddon", "upload addon", "bundle addon", "addon template".
---

# Ayon addon development

A single addon is one repo that may contain up to four sides:
server / client / frontend / services. This skill covers server + client +
frontend. For DCC hosts see `ayon-host-integration`; for services see
`ayon-events-services`.

## Repo layout

```
my_addon/
├─ package.py               # metadata — REQUIRED
├─ server/
│  ├─ __init__.py           # BaseServerAddon subclass
│  └─ settings.py           # BaseSettingsModel(s)
├─ client/
│  ├─ my_addon/             # folder name MUST match package.client_dir
│  │  ├─ __init__.py
│  │  ├─ addon.py           # AYONAddon subclass
│  │  ├─ version.py         # __version__
│  │  ├─ plugins/
│  │  │  ├─ create/  load/  publish/  inventory/
│  │  ├─ hooks/             # pre/post launch hooks
│  │  ├─ startup/           # DCC-specific boot scripts
│  │  └─ api/               # in-DCC code (keep imports lazy)
│  └─ pyproject.toml        # client deps — consumed by ayon-dependencies-tool
├─ frontend/dist/index.html # optional — built Vite/React app
├─ public/   private/       # static files
├─ services/<name>/         # dockerised services (see ayon-events-services)
├─ pyproject.toml  LICENSE  README.md
```

## `package.py` — minimum

```python
name = "my_addon"
title = "My Addon"
version = "0.0.1"
client_dir = "my_addon"          # folder under client/
```

Common extra fields in shipped addons: `ayon_server_version`,
`ayon_launcher_version`, `client_side_addon`, `services`, `plugin_for`.

## Server side — `BaseServerAddon`

Runs under the server's Python (3.12), fully async.

```python
from typing import Type
from ayon_server.addons import BaseServerAddon
from ayon_server.settings import BaseSettingsModel, SettingsField


class MySettings(BaseSettingsModel):
    enabled: bool = SettingsField(True, title="Enabled")
    studio_name: str = SettingsField(
        "", title="Studio Name", scope=["studio", "project"],
    )


DEFAULT_VALUES: dict = {}


class MyAddon(BaseServerAddon):
    settings_model: Type[MySettings] = MySettings

    async def get_default_settings(self):
        cls = self.get_settings_model()
        return cls(**DEFAULT_VALUES)

    def initialize(self):
        self.add_endpoint("studio-data", self.get_studio_data, method="GET")
        self.add_event_listener("entity.version.created", self.on_new_version)

    async def get_studio_data(self, user):
        return {"current_user": user.name}

    async def on_new_version(self, event): ...
```

### Settings — `SettingsField` kwargs

AYON-specific (source: `ayon_server/settings/settings_field.py`):

| Kwarg | Use |
|-------|-----|
| `title`, `description`, `example`, `placeholder` | UI copy |
| `scope` | `["studio", "project", "studio-site", "project-site"]` |
| `section`, `layout` | Layout hints |
| `widget`, `syntax` | `"textarea"`, `"color"`, `"password"`; `syntax` = highlighter |
| `enum_resolver` | Callable returning enum values |
| `conditional_enum` | Show only when another field matches (old `conditionalEnum` accepted) |
| `required_items` | Mark specific list entries as required |
| `tags` | UI hints (e.g. `"developer"` to hide from artists) |
| `disabled` | Render read-only |

Plus pydantic validation pass-throughs (`regex`, `min_length`, `max_length`, `ge`, `le`, `gt`, `lt`, `multiple_of`, `min_items`, `max_items`, `unique_items`, `default_factory`, `discriminator`, etc.).

Profile pattern (used all over `ayon-core`): a `list[Profile]` field where
each `Profile` carries its own `hosts`, `task_types`, `product_types`
selectors; plugins pick the right profile at runtime.

### Custom REST endpoints

Call shape: `{verb} {server}/api/addons/{name}/{version}/{endpoint}`. Same
auth as rest of the server (bearer / `X-Api-Key`).

### Web actions (server → client CLI)

Two-step. Server declares; client implements the matching `click_wrap` command.

```python
from ayon_server.actions import (
    ActionExecutor, ExecuteResponseModel, SimpleActionManifest,
)

async def get_simple_actions(self) -> list[SimpleActionManifest]:
    return [SimpleActionManifest(
        identifier="myaddon.launch.show_dialog",
        label="Show Dialog",
        icon={"type": "material-symbols", "name": "switch_access_2"},
        entity_type="folder",
    )]

async def execute_action(self, executor: ActionExecutor):
    if executor.identifier == "myaddon.launch.show_dialog":
        return await executor.get_launcher_action_response(
            args=["addon", "my_addon", "show-selected-path",
                  "--project", executor.context.project_name,
                  "--entity-id", executor.context.entity_ids[0]],
        )
```

### Frontend scopes

```python
class MyAddon(BaseServerAddon):
    frontend_scopes = {"settings": {}}     # "project", "dashboard" also supported
```

`frontend/dist/index.html` is embedded as iframe. Use
`@ynput/ayon-react-addon-provider` + `@ynput/ayon-react-components` +
styled-components + axios inside a Vite + React project.

### Static files

- `public/<path>` → `{server}/addons/{name}/{version}/public/<path>` (no auth)
- `private/<path>` → same path but requires bearer / API key
- From Python client: `ayon_api.download_addon_private_file(addon_name, addon_version, filename, destination_dir)`

### Lifecycle hooks on `BaseServerAddon`

Order (verified against `ayon-backend/develop` 2026-04-21):
`__init__` → `initialize` → `pre_setup` → `setup`. `pre_setup` and
`setup` each run once per addon version during server lifespan
startup, sequentially across all addons (all `pre_setup`s first, then
all `setup`s). `on_settings_changed(old, new, variant, project_name,
site_id, user_name)` fires later when settings are saved with
`send_event=True`. **No `post_setup`** on current `develop`.

`initialize` is called at the end of `__init__`, so it's the
canonical place to `add_endpoint` / `add_event_listener`. Failure in
`pre_setup` / `setup` unloads that addon version only; other addons
keep loading. Call `self.request_server_restart()` to force a restart.

Keep `server/` **stateless** — persistence belongs in Postgres, not
on self.

## Client side — `AYONAddon`

Runs under the launcher's Python (3.9.x VFX-platform) and, after host
install, inside DCC processes.

```python
from ayon_core.addon import AYONAddon, ITrayAddon, click_wrap
from .version import __version__


class MyAddon(AYONAddon, ITrayAddon):
    name = "my_addon"           # MUST match package.name
    label = "My Addon"
    version = __version__

    def initialize(self, settings):
        self.enabled = settings.get("enabled", True)

    def get_plugin_paths(self):
        return {
            "create":    ["/abs/path/to/create"],
            "load":      ["/abs/path/to/load"],
            "publish":   ["/abs/path/to/publish"],
            "inventory": ["/abs/path/to/inventory"],
        }

    # ITrayAddon ----------------------------------------------------------
    def tray_init(self): ...
    def tray_start(self): ...
    def tray_exit(self): ...
    def tray_menu(self, tray_menu): ...   # add QMenu entries

    # CLI -----------------------------------------------------------------
    def cli(self, click_group):
        click_group.add_command(cli_main.to_click_obj())


@click_wrap.group(MyAddon.name, help="My addon commands")
def cli_main(): pass


@cli_main.command()
@click_wrap.option("--project", type=str, required=False)
def show_selected_path(project):
    from qtpy import QtWidgets
    QtWidgets.QMessageBox.information(None, "Hi", f"{project}")
```

Invoke client CLI: `ayon addon my_addon show-selected-path --project foo`.

### Launch hooks

Live in `client/<name>/hooks/`. Pre-launch modifies subprocess env/args;
post-launch runs after start. Typical `PreLaunchHook`:

```python
from ayon_applications import PreLaunchHook

class InjectFlag(PreLaunchHook):
    app_groups = {"maya"}     # or hosts = {"maya"}
    order = 10

    def execute(self):
        self.launch_context.env["MY_FLAG"] = "1"
```

`ayon-applications` owns the `Application` / `ApplicationManager` and the
launch pipeline.

### Useful imports inside DCC context

```python
from ayon_core.pipeline import registered_host
from ayon_core.pipeline.context_tools import get_current_context, change_current_context
from ayon_core.lib import Logger
```

## Required files checklist

- [ ] `package.py` — `name`, `title`, `version`, `client_dir`
- [ ] `server/__init__.py` — `BaseServerAddon` subclass
- [ ] `server/settings.py` — at least an empty `BaseSettingsModel`
- [ ] `client/<name>/__init__.py` — `AYONAddon` subclass
- [ ] `client/<name>/version.py` — `__version__`
- [ ] `client/pyproject.toml` — client-side deps
- [ ] Build zip → upload to server → create bundle → activate

## Scaffolding

Fastest start: clone `ynput/ayon-addon-template`. It bundles
`create_package.py`, `.github/workflows/`, and `ruff.toml`.

## Tips / traps

- Never `import <dcc>` at module top level in client code — only inside
  function bodies. Launcher must load the addon even on machines without
  the DCC.
- Name matching matters: `package.name` == `AYONAddon.name` ==
  `client_dir` folder name.
- Settings UI changes can require bumping addon version and reuploading;
  value-only changes usually don't.
- The server reloads addons on file change (in dev). The launcher does not
  — restart it after client changes.

## Deeper reading

- `04-addon-structure.md` — full repo layout + `package.py` extras
- `05-server-addon.md` — settings widgets, lifecycle, patterns
- `06-client-addon.md` — all `AYONAddon` interfaces, launch hooks
- Skill `ayon-recipes` — concrete "I want to build X" patterns

## Sources

- <https://docs.ayon.dev/docs/dev_addon_creation>
- <https://docs.ayon.dev/docs/dev_addon_intro>
- <https://docs.ayon.dev/docs/dev_host_implementation> — launch hooks
- <https://help.ayon.app/articles/3005275-core-addon-settings> — profile patterns

### Source repos

- <https://github.com/ynput/ayon-addon-template> — boilerplate with `create_package.py`, CI, `ruff.toml`
- <https://github.com/ynput/ayon-backend/tree/develop/ayon_server/addons> — `BaseServerAddon` source (server side)
- <https://github.com/ynput/ayon-backend/tree/develop/ayon_server/settings> — `SettingsField`, `BaseSettingsModel`
- <https://github.com/ynput/ayon-backend/tree/develop/ayon_server/actions> — `ActionExecutor`, `SimpleActionManifest` (web actions)
- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/addon> — `AYONAddon`, `ITrayAddon`, `IPluginPaths`, `click_wrap` (client side)
- <https://github.com/ynput/ayon-applications> — `PreLaunchHook`, `Application`, `ApplicationManager`
- <https://github.com/ynput/ayon-dependencies-tool> — builds bundle dep packages
