# 04 — Addon structure

## Canonical repo layout

```
my_addon/
├─ package.py              # metadata — REQUIRED
├─ server/                 # server side code (loaded by ayon-backend)
│  ├─ __init__.py          # BaseServerAddon subclass
│  └─ settings.py          # BaseSettingsModel(s)
├─ client/
│  ├─ my_addon/            # Python pkg — folder name MUST match `client_dir`
│  │  ├─ __init__.py
│  │  ├─ addon.py          # AYONAddon subclass
│  │  ├─ version.py        # __version__ (auto-generated from package.py)
│  │  └─ plugins/
│  │     ├─ create/        # Creator plugins
│  │     ├─ load/          # Loader plugins
│  │     ├─ publish/       # pyblish plugins (collect/validate/extract/integrate)
│  │     └─ inventory/     # scene inventory actions
│  └─ pyproject.toml       # client-side deps; read by ayon-dependencies-tool
├─ frontend/
│  └─ dist/
│     └─ index.html        # built Vite/React app — embedded as iframe
├─ public/                 # static files served without auth
├─ private/                # static files behind server auth
├─ services/               # dockerised services run by ASH
│  └─ <service_name>/
│     ├─ Dockerfile
│     └─ service.py
├─ pyproject.toml          # dev-time deps / tooling
├─ LICENSE
└─ README.md
```

Not every addon uses every directory.

## `package.py`

Minimal:

```python
name = "my_addon"
title = "My Addon"
version = "0.0.1"
client_dir = "my_addon"          # folder under client/
```

Extended fields seen in official addons:

- `ayon_server_version` / `ayon_launcher_version` — required ranges
- `client_side_addon` — bool
- `services` — dict describing services exposed (name, image, env)
- `plugin_for` — list of other addons this extends

`version` must be semver-ish; the dependencies tool pins against it.

## Two halves, one repo

- **Server code** is uploaded to the server inside a zip; the server loads it
  *in-process*. Server code runs under the server's Python, not the launcher's.
- **Client code** is shipped from the server to each launcher when the bundle
  specifying it is activated. Runs inside `ayon-launcher` and, after host
  install, inside the DCC process.

They share almost no runtime but should share *concepts* (settings shape
especially — the client usually reads settings the server defines).

## Frontend

If `frontend/` is present and the server addon declares
`frontend_scopes = {"settings": {}}` (or `"project"`, `"dashboard"`, …), the
server UI will embed `/frontend/dist/index.html` as an iframe inside the named
scope.

Build setup is Vite + React:

```bash
npm create vite@latest
npm i @ynput/ayon-react-addon-provider @ynput/ayon-react-components styled-components axios
```

The host page postMessages context to the iframe (auth token, project, etc.) —
`@ynput/ayon-react-addon-provider` wraps this.

A raw `index.html` also works as long as it listens for `window.onmessage`.

## Static files — public / private

- `public/<path>` → `http://server/addons/<name>/<version>/public/<path>` — no
  auth required.
- `private/<path>` → same URL under `/private/` but requires a valid bearer
  or API key.
- From Python client:

```python
import ayon_api
ayon_api.download_addon_private_file(
    addon_name="houdini",
    addon_version="0.3.1",
    filename="client.zip",
    destination_dir="path/to/save",
)
```

Useful for shipping large binaries (luts, plugin binaries, scene templates) that
shouldn't live inside the client Python package.

## Services directory

Each subdir of `services/` is a service definition that ASH can run. Common
pattern:

- `Dockerfile` producing a small image
- `service.py` that polls `[POST] /api/enroll` (see `09-events-services.md`)
- Reads `AYON_SERVER_URL` and `AYON_API_KEY` from env provided by ASH.

## Packaging + distribution

- CI zips the addon and uploads to the server via the addons API.
- Server keeps multiple versions of an addon available concurrently (versions
  are namespaced by addon name + version string).
- Bundles pin exact versions (see `02-architecture.md#bundles`).
- The `ayon-dependencies-tool` repo is the canonical way to produce a Python
  dependency package for a bundle — it resolves every `client/pyproject.toml`
  across all addons in the bundle.

## Minimal addon checklist

- [ ] `package.py` with `name`, `title`, `version`, `client_dir`
- [ ] `server/__init__.py` with a `BaseServerAddon` subclass
- [ ] `server/settings.py` with at least an empty `BaseSettingsModel`
- [ ] `client/<name>/__init__.py` with an `AYONAddon` subclass
- [ ] `client/<name>/version.py` with `__version__`
- [ ] `client/pyproject.toml` declaring deps
- [ ] Upload zip → create/update bundle → activate bundle

## Sources

- <https://docs.ayon.dev/docs/dev_addon_creation> — full layout, `package.py`, public/private, services, frontend
- <https://docs.ayon.dev/docs/dev_addon_intro> — addon intro
- <https://github.com/ynput/ayon-addon-template> — official boilerplate (`create_package.py`, `tools/`, `.github/workflows/`, `ruff.toml`)
- <https://github.com/ynput/ayon-dependencies-tool> — builds Python dependency packages from every addon's `client/pyproject.toml`
