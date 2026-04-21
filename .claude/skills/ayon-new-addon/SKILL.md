---
name: ayon-new-addon
description: Scaffold a new Ayon addon from `ynput/ayon-addon-template`. Creates the local skeleton with `package.py`, `server/`, `client/<name>/`, fills placeholder metadata, and prints the upload-to-server + bundle-activation steps. Use when starting a new addon — saves 20 minutes of boilerplate.
when_to_use: User types `/ayon-new-addon <name>` or asks to "scaffold a new Ayon addon" / "start a new addon called X".
argument-hint: [addon_name]
disable-model-invocation: true
allowed-tools: Bash(git clone *) Bash(cp *) Bash(mv *) Bash(rm *) Bash(ls *) Bash(sed *) Bash(find *) Bash(mkdir *) Read Write Edit
---

# Scaffold a new Ayon addon

Target addon name: **$ARGUMENTS**

## Preconditions

Before touching anything, confirm with the user:

1. **Where** should the addon repo go? Ask for target directory if not
   already clear from context (e.g. `~/Git/ayon-my_addon`).
2. **Name validity**: snake_case, no spaces, starts with a letter, no
   collision with an existing `ynput/ayon-*` repo. If in doubt,
   `gh repo view ynput/ayon-<name>` to check.
3. **Intent**: is this a pure client addon, server-only, has frontend, has
   services? This shapes what to prune from the template.
4. **Title** (human-readable label), **initial version** (default `0.1.0`),
   and **client_dir** (default: the same as the name).

Wait for answers before running commands.

## Playbook

### Step 1 — clone the template

```bash
git clone https://github.com/ynput/ayon-addon-template.git <target-dir>
cd <target-dir>
rm -rf .git
```

Re-init as a fresh repo **only if the user asks** (they may want to
inspect before committing).

### Step 2 — rename the client package

The template has `client/<placeholder>/` — rename to `client/$ARGUMENTS/`.
Also update any imports (`from <placeholder> import ...`).

```bash
find client -maxdepth 1 -type d -name "<placeholder>" -exec mv {} client/$ARGUMENTS \;
```

Check `client/<addon_name>/__init__.py` and `client/<addon_name>/addon.py`
(or `.version.py`) for literal placeholder strings.

### Step 3 — fill `package.py`

```python
name = "$ARGUMENTS"
title = "<ask user>"
version = "0.1.0"
client_dir = "$ARGUMENTS"
```

Also set `ayon_server_version` and `ayon_launcher_version` if known;
leave blank for "any supported" otherwise.

### Step 4 — minimum `server/__init__.py`

```python
from typing import Type
from ayon_server.addons import BaseServerAddon
from ayon_server.settings import BaseSettingsModel, SettingsField


class Settings(BaseSettingsModel):
    enabled: bool = SettingsField(True, title="Enabled")


class MyAddon(BaseServerAddon):
    settings_model: Type[Settings] = Settings

    async def get_default_settings(self):
        return self.get_settings_model()()
```

Rename `MyAddon` to match the user's title (PascalCase of `$ARGUMENTS`).

### Step 5 — minimum `client/$ARGUMENTS/__init__.py`

```python
from .addon import MyAddon

__all__ = ["MyAddon"]
```

`client/$ARGUMENTS/addon.py`:

```python
from ayon_core.addon import AYONAddon
from .version import __version__


class MyAddon(AYONAddon):
    name = "$ARGUMENTS"
    label = "<title>"
    version = __version__

    def initialize(self, settings):
        self.enabled = settings.get("enabled", True)
```

`client/$ARGUMENTS/version.py`:

```python
__version__ = "0.1.0"
```

### Step 6 — prune unused directories

Ask the user which sides the addon needs:

| Side | Dirs to keep |
|------|--------------|
| Server only | `server/`, `package.py`, root `pyproject.toml` |
| Client only | `client/`, `package.py` |
| With frontend | keep `frontend/` (Vite project); bootstrap with `npm create vite@latest` inside |
| With services | keep `services/<name>/`; add Dockerfile + service.py |
| With static files | keep `public/` and/or `private/` |

Delete what isn't needed. Don't delete `package.py`, `pyproject.toml`,
`ruff.toml`, `LICENSE`, `README.md`, `.github/workflows/`, or
`create_package.py`.

### Step 7 — dependencies

Update `client/pyproject.toml` with the addon's client Python deps. Keep
it minimal — added deps inflate the bundle's dep package.

### Step 8 — first zip + upload

```bash
python create_package.py
```

Then upload:

```bash
curl -X POST "$AYON_SERVER_URL/api/addons" \
  -H "X-Api-Key: $AYON_API_KEY" \
  --data-binary "@package/$ARGUMENTS-0.1.0.zip"
```

(Or via the web UI: Studio Settings → Bundles → upload.)

### Step 9 — create a dev bundle

Server UI → Bundles → **Create dev bundle** → assign to your user →
include `$ARGUMENTS` at version `0.1.0` → (optional) point its local
path at the cloned repo's `client/` dir so edits reload.

### Step 10 — test

```bash
ayon --use-dev
# then:
ayon addon $ARGUMENTS --help      # if you added a CLI
```

Verify the addon shows in Studio Settings.

## Final output to the user

After running through the playbook, report:

- Directory created: `<path>`
- Files populated: `package.py`, `server/__init__.py`,
  `client/$ARGUMENTS/{__init__,addon,version}.py`
- Directories removed (if any)
- Next step: upload zip + create dev bundle
- Pointer: Skill `ayon-addon-development` for adding settings, endpoints,
  tray widgets, hooks, etc. Skill `ayon-recipes` for common extensions.

## Notes

- **Don't re-init git** without asking.
- **Don't push anywhere** — user decides where this lives.
- If the user already has a partial addon in the target dir, **stop** and
  ask before overwriting.
