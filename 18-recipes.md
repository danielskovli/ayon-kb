# 18 — Developer recipes

Concrete "I want to build X → do Y" patterns, each pointing at the
relevant KB file. These are scaffolds, not finished code — grep the named
base class in `ayon-core` for the current signature before relying on it.

## 1. "I want to create a brand-new addon from scratch"

1. Clone `github.com/ynput/ayon-addon-template` as a starting point.
2. Edit `package.py`: set `name`, `title`, `version`, `client_dir`.
3. Flesh out `server/__init__.py` with a `BaseServerAddon` subclass — start
   with just `settings_model = EmptySettings` (see `05-server-addon.md`).
4. Flesh out `client/<name>/__init__.py` with `AYONAddon` + `name` (see
   `06-client-addon.md`).
5. `python create_package.py` → produces the upload zip.
6. Server UI → Bundles → upload zip → create a **dev bundle** pinned to
   your user (see `11-launcher-dev.md#dev-mode`).
7. `ayon --use-dev`. Verify the addon shows up in Studio Settings and, if
   it's a client addon, that `ayon addon <name> --help` responds.

## 2. "I want a button on the server UI that runs a CLI command on the artist's machine"

- **Server side**: implement `get_simple_actions()` and `execute_action()`
  returning `executor.get_launcher_action_response(args=[...])`. See
  `05-server-addon.md#web-actions`.
- **Client side**: expose a `click_wrap` command under your
  `AYONAddon.cli()`. See `06-client-addon.md#cli--click_wrap`.
- Args in `execute_action` become argv on the client.
- The icon uses material-symbols glyph names — `{"type": "material-symbols",
  "name": "rocket_launch"}`.

## 3. "I want to react to an event on the server"

Two options depending on blast-radius:

- **In-process listener** (OK for lightweight reactions, same-tx with the
  event): `BaseServerAddon.initialize()` → `add_event_listener(topic, fn)`.
  See `05-server-addon.md#event-handlers`.
- **External service** (OK for long-running work, survives restarts): use
  the enroll pattern with an ASH-run container. See
  `09-events-services.md#the-enroll-pattern-services`.

Rule of thumb: HTTP-handler-lifetime work → listener; >30s / external
calls / rate-limit sensitive → service.

## 4. "I want to publish a new product type from Maya"

1. Pick a `productType` name — lowercase, reused across DCCs if possible.
   Common types are in `help.ayon.app/en/articles/7070980-about-ayon-pipeline`.
2. Add the type to Project Anatomy → product types.
3. In `ayon-maya/client/ayon_maya/plugins/create/`, add a `Creator`:
   - `family = "<productType>"`
   - `get_instance_attr_defs` — the knobs artists will see.
   - `create()` — build the scene metadata (usually an objectSet).
4. In `ayon-core/.../publish/` or your addon, add:
   - `CollectX` (`order = CollectorOrder`) — fills instance data.
   - `ValidateX` (`order = ValidatorOrder`) — checks scene integrity.
   - `ExtractX` (`order = ExtractorOrder`) — writes files to staging.
   - The Integrator in `ayon-core` usually handles registration.
5. Add a matching `LoaderPlugin` in `ayon-<host>/.../load/` so artists can
   load the new type. See `15-loaders-inventory.md`.

## 5. "I want a validator that repairs itself"

Pyblish `Action`s attached to the plugin provide repair buttons in the
Publisher.

```python
import pyblish.api


class RepairPivot(pyblish.api.Action):
    label  = "Reset pivot to origin"
    on     = "failed"   # show only after failure
    icon   = "wrench"

    def process(self, context, plugin):
        for instance in context:
            ...  # mutate the scene


class ValidatePivot(pyblish.api.InstancePlugin):
    order   = pyblish.api.ValidatorOrder
    hosts   = ["maya"]
    families = ["model"]
    actions = [RepairPivot]

    def process(self, instance):
        if instance.data.get("pivot_off_origin"):
            raise PublishValidationError(
                "Pivot is not at origin",
                title="Pivot",
                description="Model must have pivot at origin. Use *Repair*.",
            )
```

See `08-publishing.md#validation-errors`.

## 6. "I want to download representations to disk outside a DCC"

```python
import ayon_api
from ayon_core.pipeline.load import (
    get_representation_by_names, get_representation_path,
)
from ayon_core.pipeline.anatomy import Anatomy

project = "my_project"
repre = get_representation_by_names(
    project, "/seq010/sh010", "modelMain", "v003", "abc"
)
path = get_representation_path(project, repre, anatomy=Anatomy(project))
print(path)
```

For Site-Sync-aware downloads, call the site-sync addon's API instead of
relying on the path being resolved locally.

## 7. "I want to add a new scope to the Ayon web UI"

In the server addon: `frontend_scopes = {"project": {}}` (or `"settings"`,
`"dashboard"`). Place the built Vite app in `frontend/dist/`. See
`05-server-addon.md#frontend-scopes` + `04-addon-structure.md#frontend`.

Bootstrap a project:
```bash
npm create vite@latest
npm i @ynput/ayon-react-addon-provider @ynput/ayon-react-components styled-components axios
```

`@ynput/ayon-react-addon-provider` wraps the iframe `postMessage`
handshake so you can use `useAddonContext()`-style hooks.

## 8. "I want to expose a custom REST endpoint"

Add a method on your `BaseServerAddon` and wire it in `initialize()`:

```python
def initialize(self):
    self.add_endpoint("stats", self.get_stats, method="GET")

async def get_stats(self, user: CurrentUser):
    return {"by_me": await count_things_for(user)}
```

Call shape: `GET {server}/api/addons/{name}/{version}/stats`. Bearer or
API-key auth applies. See `05-server-addon.md#rest-endpoints`.

## 9. "I want a settings field that only shows for some hosts"

Pattern used throughout `ayon-core`: a `list[Profile]` field where each
profile carries its own `hosts`, `task_types`, `product_types` filters. The
plugin (creator / publisher / loader) calls a shared filter function on
settings to pick the profile for the current context.

See `05-server-addon.md#settings-ui--widgets--patterns` and grep
`filter_profiles` in `ayon-core`.

## 10. "I want to package + ship my addon via CI"

The `ayon-addon-template` includes `.github/workflows/` examples. The core
steps are:

1. `python create_package.py` — builds the zip.
2. `curl -X POST $AYON_SERVER/api/addons -H "X-Api-Key: …" --data-binary @package.zip`
   (or `ayon_api.upload_addon(path)` if present in your version).
3. (Optionally) auto-create or update a staging bundle pinned to the new
   version via the bundles API.

`ayon-dependencies-tool` handles the dep-package side of the bundle.

## 11. "I want to run a one-off script in the current DCC's env"

From any running host process:

```python
from ayon_core.pipeline import registered_host
from ayon_core.pipeline.context_tools import get_current_context

host = registered_host()
ctx  = get_current_context()
print(ctx)  # {project_name, folder_path, task_name}
```

For scripts outside a DCC, talk to the server directly via `ayon_api` —
see `10-apis.md`.

## 12. "I want to inject env vars into a DCC launch"

`PreLaunchHook` in your addon's `hooks/` folder. Filter with `app_groups` /
`hosts`. See `06-client-addon.md#launch-hooks`.

## 13. "I want to validate colorspace on publish"

Make your extractor inherit `ColormanagedPyblishPluginMixin` and call
`set_representation_colorspace(repre, colorspace=...)` before appending to
`instance.data["representations"]`. See
`16-site-sync-colorspace.md#extractor-side`.

## 14. "I want to run the same plugin on the farm (Deadline)"

The `ayon-deadline` addon wraps publish plugins with farm submitters.
Pattern:

- The artist-side creator adds a **farm version** of the product (e.g.
  `renderFarmMain`).
- A publish plugin (`SubmitPublishJob` / family-specific submitter) serialises
  the instance and submits a Deadline job.
- The farm job runs `ayon` with the right bundle, loads the submitted
  context, runs the same extractors remotely.

See `ayon-deadline` source and `docs.ayon.dev/docs/dev_deadline`.

## 15. "I want to test my addon locally"

- Dev mode with `--use-dev` and a user-pinned dev bundle is the primary
  workflow (see `11-launcher-dev.md#dev-mode`).
- For publish-plugin logic, copy the pattern in `ayon-core/tests/` — spin
  up a disposable Mongo, run `start.py runtests`, assert DB + files.
- For server addon logic, unit-test against a disposable Postgres; there
  isn't (yet) an official integration harness for server addons separate
  from the full server test suite.

## Sources

Each recipe references the relevant in-KB file for deeper context. The
KB files cite their own sources — no new URLs here.
