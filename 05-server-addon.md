# 05 — Server-side addon

The **server half** of an addon lives under `server/` and is loaded by
`ayon-backend` at startup. It runs under the server's Python interpreter (3.12)
and has full access to the server's async machinery.

## Minimal `server/__init__.py`

```python
from typing import Type
from ayon_server.addons import BaseServerAddon
from ayon_server.settings import BaseSettingsModel


class MySettings(BaseSettingsModel):
    """Settings schema for this addon."""
    pass


DEFAULT_VALUES: dict = {}


class MyAddon(BaseServerAddon):
    settings_model: Type[MySettings] = MySettings

    async def get_default_settings(self):
        cls = self.get_settings_model()
        return cls(**DEFAULT_VALUES)
```

That alone is enough to show up in the server's addon list and get a settings
pane in the web UI.

## Settings

```python
from ayon_server.settings import BaseSettingsModel, SettingsField


class MySettings(BaseSettingsModel):
    enabled: bool = SettingsField(True, title="Enabled")
    studio_name: str = SettingsField(
        "",
        title="Studio Name",
        scope=["studio", "project"],
    )
```

- Settings are strongly typed via pydantic.
- `scope` controls where overrides are allowed. Values: `studio`, `project`,
  `studio-site`, `project-site`. Omit for ubiquitous.
- Nested submodels are fine; they become collapsible sections in the UI.
- Common field types: `bool`, `int`, `float`, `str`, `list[str]`, `list[Model]`.
- Special UI hints: `section=""`, `layout="expanded"`, widgets via
  `widget="textarea"`, etc. (check `ayon_server.settings` for the full list).

## REST endpoints

Custom endpoints are registered from `initialize()`:

```python
class MyAddon(BaseServerAddon):

    def initialize(self):
        self.add_endpoint(
            "studio-data",
            self.get_studio_data,
            method="GET",
        )

    async def get_studio_data(self, user: CurrentUser):
        return {
            "secret": "there is no secret",
            "current_user": user.name,
        }
```

Call shape:

```
GET {server}/api/addons/{addon_name}/{addon_version}/studio-data
```

`CurrentUser`, `CurrentUserOptional` and request-dependency helpers live under
`ayon_server`. For path params / query models follow FastAPI conventions.

## Event handlers

Register async handlers from `initialize()` (or `setup()`):

```python
async def on_version_created(event):
    ...

def initialize(self):
    self.add_event_listener("entity.version.created", on_version_created)
```

See `09-events-services.md` for the event taxonomy.

## Web (UI) actions

"Web actions" let the server UI trigger a CLI command on a specific client
machine (via the launcher). Two-step: server declares, client implements.

**Server side:**

```python
from ayon_server.actions import (
    ActionExecutor,
    ExecuteResponseModel,
    SimpleActionManifest,
)


async def get_simple_actions(self) -> list[SimpleActionManifest]:
    return [
        SimpleActionManifest(
            identifier="myaddon.launch.show_dialog",
            label="Show Dialog",
            icon={"type": "material-symbols", "name": "switch_access_2"},
            entity_type="folder",
        ),
    ]


async def execute_action(self, executor: ActionExecutor) -> ExecuteResponseModel:
    if executor.identifier == "myaddon.launch.show_dialog":
        return await executor.get_launcher_action_response(
            args=[
                "addon", "my_addon", "show-selected-path",
                "--project", executor.context.project_name,
                "--entity-id", executor.context.entity_ids[0],
            ]
        )
```

**Client side** (CLI command triggered via launcher — see `06-client-addon.md`).

## Frontend scopes

Declaring a frontend mounts `frontend/dist/index.html` into the Ayon UI:

```python
from typing import Any

class MyAddon(BaseServerAddon):
    frontend_scopes: dict[str, Any] = {"settings": {}}
```

Known scopes:
- `"settings"` — appears in the addon's settings pane (next to auto-generated form)
- `"project"` — a tab inside a project
- `"dashboard"` — top-level tab

## Static files

Dropped in `public/` (no auth) or `private/` (auth required).

- `GET {server}/addons/{name}/{version}/public/{path}`
- `GET {server}/addons/{name}/{version}/private/{path}` (bearer or API key)

## Lifecycle hooks on `BaseServerAddon`

Commonly overridden async methods:

| Method | When |
|--------|------|
| `setup` | One-time on addon load |
| `initialize` | After `setup`, wire endpoints + event listeners |
| `get_default_settings` | UI reset / project init |
| `pre_setup` / `post_setup` | Ordered hooks around initialization |
| `on_settings_changed` | When studio/project overrides change |

Check `ayon_server.addons.base` for the full surface — it evolves.

## Tips

- Keep `server/` **stateless**. Persistence belongs in the DB, not in addon
  instance attributes.
- Use `ayon_server.lib.postgres.Postgres.execute()` for DB calls; don't open
  your own pool.
- For long-running work, emit a persistent event and let a **service** (via
  ASH) do it — don't block an HTTP request.
- Log via `from ayon_server.logging import log`.
- Use `scope` on settings aggressively — it's the single best way to keep
  projects independent.

## Settings UI — widgets & patterns

`SettingsField` accepts extra kwargs that control the rendered UI. Seen in
official addons (check the current `ayon_server.settings` source for the full
list):

| Kwarg | Purpose |
|-------|---------|
| `title` | Label shown in UI |
| `description` | Help tooltip (supports short markdown) |
| `example` | Placeholder / example value |
| `scope` | `["studio","project","studio-site","project-site"]` |
| `section` | Start a new collapsible section above this field |
| `layout` | `"compact"`, `"expanded"` — override default |
| `widget` | Override control (`"textarea"` for `str`, `"color"`, `"password"`) |
| `conditional_enum` | Show only when another field equals a given value |
| `tags` | UI hint tags (e.g. `"developer"` to hide from artists) |
| `regex` / `min_length` / `max_length` | Validation on `str` |
| `ge` / `le` / `gt` / `lt` | Validation on numbers |

Common patterns:

```python
from ayon_server.settings import BaseSettingsModel, SettingsField, MultiplatformStr


class Roots(BaseSettingsModel):
    """Studio roots — platform-specific paths."""
    work: MultiplatformStr = SettingsField(default_factory=MultiplatformStr,
                                           title="Work root")


class MySettings(BaseSettingsModel):
    enabled: bool = SettingsField(True, title="Enabled")
    mode: str = SettingsField(
        "auto", title="Mode",
        enum_resolver=lambda: ["auto", "manual", "off"],
    )
    farm_settings: "FarmSettings" = SettingsField(
        default_factory=lambda: FarmSettings(),
        title="Farm", section="Rendering",
    )
    tags: list[str] = SettingsField(default_factory=list, title="Tags")
```

For **per-profile/per-host** settings (filter by host + task type), addons
typically use list-of-model fields whose items contain `hosts`, `task_types`,
`product_types` selectors. Grep `apply_settings` in `ayon-core` for the
canonical matching code.

## Sources

- <https://docs.ayon.dev/docs/dev_addon_creation> — `BaseServerAddon`, settings model, endpoints, web actions, frontend scopes, static files
- <https://docs.ayon.dev/docs/dev_event_system> — event listeners
- <https://help.ayon.app/articles/3005275-core-addon-settings> — real-world examples of profile-based settings (Extract Review, Integrate Product Group, etc.)
- `ayon_server.addons.BaseServerAddon` source — lifecycle hooks (`setup`, `initialize`, `pre_setup`, `post_setup`, `on_settings_changed`)
- `ayon_server.settings.SettingsField` source — widget kwargs (the list above is synthesised from patterns seen in `ayon-core`; verify against live source)
