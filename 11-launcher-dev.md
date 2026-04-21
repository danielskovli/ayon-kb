# 11 — Launcher, bundles, dev mode, testing

## Launcher essentials

### Executables

- Windows: `ayon.exe` (silent) / `ayon_console.exe` (with console output)
- Linux/macOS: `ayon`

### CLI flags

| Flag | Meaning |
|------|---------|
| `init-ayon-launcher` | One-time OS registration (shim install) |
| `--bundle <NAME>` | Force a specific bundle |
| `--verbose <LEVEL>` | `DEBUG` / `INFO` / `WARNING` / `ERROR` / `CRITICAL` |
| `--debug` | Shorthand for `--verbose DEBUG` |
| `--skip-headers` | Drop console banner |
| `--use-dev` | Use assigned dev bundle |
| `--use-staging` | Use staging bundle |
| `--headless` | No UI during bootstrap |
| `--ayon-login` | Force login dialog |
| `--skip-bootstrap` | Skip bootstrap (for manual setups) |

Example:
```
./ayon.exe --bundle my-bundle --verbose DEBUG
```

### Environment variables

| Var | Purpose |
|-----|---------|
| `AYON_SERVER_URL` | Connection target |
| `AYON_API_KEY` | Service account key |
| `AYON_VERSION` | Active launcher version |
| `AYON_BUNDLE_NAME` | Active bundle name |
| `AYON_LAUNCHER_STORAGE_DIR` | Addons + deps cache |
| `AYON_LAUNCHER_LOCAL_DIR` | Per-machine user files |
| `AYON_DEBUG` | `1` = debug |
| `AYON_HEADLESS_MODE` | `1` = headless |
| `AYON_USE_DEV` | `1` = dev mode |
| `AYON_PROJECT_NAME`, `AYON_FOLDER_PATH`, `AYON_TASK_NAME` | Current context (set before DCC launch) |

### Storage dirs (defaults)

- Windows: `%LOCALAPPDATA%\Ynput\AYON`
- Linux: `~/.local/share/AYON`
- macOS: `~/Library/Application Support/AYON`

### Running from source

```
poetry run python start.py [args...]
```

The Windows dev helper is `tools/ayon_console.bat --use-dev ...`.

### Uploading a custom launcher build

`./tools/manage.ps1 upload --server <url> --api-key <key>`  (Windows)
`./tools/make.sh  upload --server <url> --api-key <key>`    (Linux/macOS)

The server then distributes it to workstations that use a bundle pinned to
that version.

## Bundles

Declared server-side (`/bundles` API / web UI). A bundle pins:

- launcher version
- Python dependency package version
- addon name → version map
- flag: `isProduction` / `isStaging` / `isDev`
- (dev bundles only) assigned user + optional `addonDevelopment` map of
  `addon_name → local_path`

The launcher resolves:

- default → production bundle
- `--use-staging` → staging bundle
- `--use-dev` → the dev bundle belonging to the logged-in user

## Dev mode

Minimums (per the docs):
- Ayon server ≥ 0.5.0
- Ayon launcher ≥ 1.0.0-beta.6
- OpenPype addon ≥ 3.17.3

Steps:

1. Admin marks the user as **Developer** in the Web UI.
2. User refreshes → sees "Developer mode" checkbox top-right; enables it.
3. User creates a dev bundle in the Bundles page, assigns themselves, and
   (optionally) maps addons to local paths.
4. User launches with `ayon --use-dev` (or sets `AYON_USE_DEV=1` +
   `AYON_BUNDLE_NAME`).
5. Edit local code; restart launcher; changes apply.

**Gotcha**: custom addon paths **only affect client code**. Server-side
changes still require re-uploading the addon (and bumping its version, or
overwriting if the server allows).

## Dependencies tool

`ayon-dependencies-tool` builds a **dep package** — a zip of wheels for
`ayon-core`'s Python + every addon's `client/pyproject.toml`. Each bundle
pins a dep package so all workstations use the same libs.

When you bump an addon's client deps, you typically:

1. Merge the addon PR.
2. Rebuild the dep package.
3. Create a new bundle (or new staging bundle) using the new dep package +
   new addon version.

## Testing

Test layout (`ayon-core` and other addons):

```
tests/
├─ integration/    # end-to-end publish tests per host
├─ unit/           # per-module unit tests
├─ lib/            # helpers (db_handler, file_handler, assert_classes,
│                  #         testing_classes)
└─ resources/      # fixture data
```

**Integration tests** currently run in legacy "OpenPype mode" (MongoDB). They
require `mongorestore`, `mongodump`, `mongoimport` on PATH.

Run them:
```
./.poetry/bin/poetry run python start.py runtests ./tests/integration/hosts/nuke
```

Notable arguments:

| Flag | Meaning |
|------|---------|
| `-m <text>` / `--mark <text>` | pytest mark filter |
| `-p <text>` / `--pyargs <text>` | run by Python package |
| `--test_data_folder <path>` | pre-unzipped fixture dir |
| `-s` / `--persist` | keep DB + files after run |
| `-a <text>` / `--app_variant <text>` | target specific DCC variant |
| `--timeout <secs>` | per-phase timeout |
| `-so` / `--setup_only` | provision DB but skip run |

Implemented hosts in integration tests: Maya, Nuke, AfterEffects, Photoshop.

Phases of a publishing integration test:

1. Prepare — download data + prep env.
2. Launch — start DCC with workfile.
3. Publish — run the publish session.
4. Validate — compare DB + filesystem to expected.
5. Cleanup — unless `--persist`.

## Contributing

From `docs.ayon.dev/docs/dev_contribute`:

- Git flow: `main` (prod), `develop` (integration — PR target), `release/3.x.x`
  (stabilisation).
- Branch naming: `feature/{issue}-{name}`, `bugfix/{issue}-{name}`,
  `hotfix/{issue}-{name}` (hotfix branches from `main`).
- PRs go to `develop` unless otherwise directed.
- New features should be discussed in Ynput Community forums **before**
  implementation.
- Commit message / lint specifics aren't formally documented — read CI config
  per repo (many use `ruff` / `black` / `isort`).

## Deploying a fresh server for dev

`ayon-docker` provides the compose file:

```bash
git clone https://github.com/ynput/ayon-docker
cd ayon-docker
make demo                # macOS/Linux, loads demo_Commercial
# or
./manage.ps1 demo        # Windows
```

Server ends up at `http://localhost:5000`. Admin credentials are printed on
first boot.

## Sources

- <https://docs.ayon.dev/docs/dev_launcher> — CLI flags, env vars, storage dirs, build (`CX_Freeze`, Poetry)
- <https://docs.ayon.dev/docs/dev_dev_mode> — dev bundle requirements (server ≥ 0.5.0, launcher ≥ 1.0.0-beta.6, OpenPype addon ≥ 3.17.3), activation flow
- <https://docs.ayon.dev/docs/dev_requirements> — Python 3.9.x, VFX Platform CY2022, OS support
- <https://docs.ayon.dev/docs/dev_testing> — test layout, integration test runner args, phases
- <https://docs.ayon.dev/docs/dev_contribute> — git flow (`main`/`develop`/`release/*`), branch naming, PR target
- <https://github.com/ynput/ayon-launcher> — launcher source + build tooling
- <https://github.com/ynput/ayon-docker> — docker-compose, demo projects
- <https://github.com/ynput/ayon-dependencies-tool> — dep-package builder
