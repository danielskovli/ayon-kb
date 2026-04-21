---
name: ayon-recipes
description: Ready-made Ayon developer recipes — concrete "I want to build X → do Y" patterns covering new addon scaffolding, server UI actions that run client CLI, event reactions, new product types, self-repairing validators, downloading representations outside a DCC, adding frontend scopes, custom REST endpoints, host-scoped settings, CI packaging, running scripts inside DCC env, launch-env injection, colorspace in publish, Deadline farm submission, local addon testing. Use when starting an implementation task that matches a common pattern.
when_to_use: Triggered by "I want to build", "how do I scaffold", "I need to react to event X", "add a button that runs on the artist's machine", "new product type", "validator with repair", "download outside DCC", "expose REST endpoint", "frontend tab in Ayon UI", "host-scoped setting", "CI for my addon", "ship to Deadline", "test my addon".
---

# Ayon developer recipes

Concrete patterns, each pointing at the relevant area. Scaffolds, not
finished code — always grep the current source before committing to a
signature.

## 1. Create a brand-new addon from scratch
1. Clone `github.com/ynput/ayon-addon-template`.
2. Edit `package.py` → `name`, `title`, `version`, `client_dir`.
3. Flesh out `server/__init__.py` (`BaseServerAddon`).
4. Flesh out `client/<name>/__init__.py` (`AYONAddon`).
5. `python create_package.py` → zip.
6. Server UI → Bundles → upload → create **dev bundle** pinned to your
   user.
7. `ayon --use-dev` → verify in Settings + `ayon addon <name> --help`.

Skills: `ayon-addon-development`, `ayon-launcher-dev`.

## 2. Server UI button that runs a CLI command on the artist's machine
- **Server**: `get_simple_actions()` + `execute_action()` returning
  `executor.get_launcher_action_response(args=[...])`.
- **Client**: `click_wrap` command registered via `AYONAddon.cli()`.
- `args` in `execute_action` → argv on the client.
- Icon: `{"type": "material-symbols", "name": "rocket_launch"}`.

Skills: `ayon-addon-development`.

## 3. React to a server event
Two options:

- **In-process listener** (same-tx, <1s work): server addon's
  `initialize()` → `add_event_listener("topic", fn)`.
- **External service** (long work, rate-limit sensitive, survives
  restarts): ASH-run service using the enroll pattern.

Skills: `ayon-events-services`.

## 4. Publish a new product type from Maya
1. Pick lowercase `productType`; add it to Project Anatomy → Product Types.
2. In `ayon-maya/client/ayon_maya/plugins/create/` — `Creator` with
   `family`, `get_instance_attr_defs`, `create()`.
3. In `ayon-core` or your addon — `CollectX`, `ValidateX`, `ExtractX`
   plugins at correct pyblish orders; the integrator in `ayon-core`
   handles registration.
4. Matching `LoaderPlugin` in `ayon-<host>/.../load/`.

Skills: `ayon-publishing`, `ayon-loaders-inventory`, `ayon-host-integration`.

## 5. Validator that self-repairs
Pyblish `Action` attached to the plugin (`on = "failed"`).

```python
class RepairPivot(pyblish.api.Action):
    label = "Reset pivot"
    on    = "failed"
    def process(self, context, plugin): ...

class ValidatePivot(pyblish.api.InstancePlugin):
    actions = [RepairPivot]
    def process(self, instance): raise PublishValidationError(...)
```

Skills: `ayon-publishing`.

## 6. Download representations to disk outside a DCC
```python
from ayon_core.pipeline.load import (
    get_representation_by_names, get_representation_path,
)
from ayon_core.pipeline.anatomy import Anatomy

repre = get_representation_by_names(project, folder_path, product_name,
                                    version_name, repre_name)
path = get_representation_path(project, repre, anatomy=Anatomy(project))
```

For Site-Sync-aware download, call the site-sync addon's API (not path
resolution alone).

Skills: `ayon-loaders-inventory`, `ayon-anatomy-templates`,
`ayon-site-sync-colorspace`, `ayon-apis`.

## 7. Add a tab to the Ayon web UI
```python
frontend_scopes = {"project": {}}   # or "settings", "dashboard"
```
Place built Vite app in `frontend/dist/`. Bootstrap:
```bash
npm create vite@latest
npm i @ynput/ayon-react-addon-provider @ynput/ayon-react-components styled-components axios
```
`@ynput/ayon-react-addon-provider` wraps the `postMessage` handshake.

Skills: `ayon-addon-development`.

## 8. Expose a custom REST endpoint
```python
def initialize(self):
    self.add_endpoint("stats", self.get_stats, method="GET")

async def get_stats(self, user: CurrentUser):
    return {"by_me": await count_things_for(user)}
```
Call: `GET {server}/api/addons/{name}/{version}/stats`.

Skills: `ayon-addon-development`, `ayon-apis`.

## 9. Settings field that only shows for some hosts
Pattern used throughout `ayon-core`: a `list[Profile]` field where each
profile carries `hosts`, `task_types`, `product_types` filters. Plugins
call a shared filter function on settings to pick the right profile.

Grep `filter_profiles` in `ayon-core`.

Skills: `ayon-addon-development`.

## 10. Package + ship an addon via CI
`ayon-addon-template` includes `.github/workflows/`. Core steps:

1. `python create_package.py` → zip
2. `curl -X POST $AYON_SERVER/api/addons -H "X-Api-Key: ..." --data-binary @package.zip`
   (or `ayon_api.upload_addon(path)` if present)
3. Optionally auto-update a staging bundle via the bundles API.

Skills: `ayon-addon-development`, `ayon-apis`.

## 11. Run a script in the current DCC's env
```python
from ayon_core.pipeline import registered_host
from ayon_core.pipeline.context_tools import get_current_context

host = registered_host()
ctx  = get_current_context()   # {project_name, folder_path, task_name}
```

Outside a DCC: talk to the server via `ayon_api`.

Skills: `ayon-host-integration`, `ayon-apis`.

## 12. Inject env vars into a DCC launch
`PreLaunchHook` in your addon's `hooks/` folder; filter with `app_groups`
or `hosts`.

Skills: `ayon-addon-development`.

## 13. Validate colorspace on publish
Extractor inherits `ColormanagedPyblishPluginMixin`; call
`set_representation_colorspace(repre, colorspace=...)` before appending
to `instance.data["representations"]`.

Skills: `ayon-site-sync-colorspace`, `ayon-publishing`.

## 14. Run the same plugin on the farm (Deadline)
Pattern via `ayon-deadline`:
- Creator adds a **farm variant** product (e.g. `renderFarmMain`).
- A publish plugin (`SubmitPublishJob` / family-specific submitter)
  serialises the instance and submits a Deadline job.
- The farm job runs `ayon` with the right bundle, reloads the submitted
  context, runs extractors remotely.

See `ayon-deadline` source and `docs.ayon.dev/docs/dev_deadline`.

Skills: `ayon-publishing`, `ayon-dcc-integrations`.

## 15. Test an in-dev addon
- Dev mode with `--use-dev` + user-pinned dev bundle is primary.
- Publish-plugin logic → copy `ayon-core/tests/` pattern, spin up
  disposable Mongo, run `start.py runtests`, assert DB + files.
- Server addon logic → unit-test against a disposable Postgres;
  integration harness for server addons is still part of the full server
  test suite.

Skills: `ayon-launcher-dev`.

## Deeper reading

- `18-recipes.md` — same list in the human-readable KB with Sources

## Sources

Each recipe defers to the KB file named in the "Skills" line. Those
files carry their own Sources blocks.
