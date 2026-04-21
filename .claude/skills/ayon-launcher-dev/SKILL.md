---
name: ayon-launcher-dev
description: Ayon launcher, bundles, developer mode, and testing — `ayon`/`ayon.exe` CLI flags, env vars (`AYON_*`), storage directories per OS, bundle concept (prod/staging/dev), dev-mode workflow (`--use-dev`), assigning a dev bundle to yourself, `ayon-dependencies-tool`, integration test runner, contributing git-flow. Use when setting up dev mode, diagnosing launcher issues, configuring a bundle, running tests, or onboarding to Ayon development.
when_to_use: Triggered by "launcher", "ayon.exe", "ayon_console", "bundle", "staging", "dev bundle", "--use-dev", "AYON_SERVER_URL", "AYON_BUNDLE_NAME", "AYON_LAUNCHER_STORAGE_DIR", "dependency package", "ayon-dependencies-tool", "ayon-docker", "integration tests", "runtests", "contributing", "feature branch".
---

# Ayon launcher, bundles, dev mode, testing

## Launcher executables

- Windows: `ayon.exe` (silent) / `ayon_console.exe` (with console output)
- Linux / macOS: `ayon`

## CLI flags

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

Example: `./ayon.exe --bundle my-bundle --verbose DEBUG`

## Environment variables

| Var | Purpose |
|-----|---------|
| `AYON_SERVER_URL` | Connection target |
| `AYON_API_KEY` | Service account key |
| `AYON_VERSION` | Active launcher version |
| `AYON_BUNDLE_NAME` | Active bundle |
| `AYON_LAUNCHER_STORAGE_DIR` | Addons + dependency cache |
| `AYON_LAUNCHER_LOCAL_DIR` | Per-machine user files |
| `AYON_DEBUG` | `1` = debug |
| `AYON_HEADLESS_MODE` | `1` = headless |
| `AYON_USE_DEV` | `1` = dev mode |
| `AYON_PROJECT_NAME`, `AYON_FOLDER_PATH`, `AYON_TASK_NAME` | Current context (set before DCC launch) |

## Storage dirs (defaults)

- Windows: `%LOCALAPPDATA%\Ynput\AYON`
- Linux: `~/.local/share/AYON`
- macOS: `~/Library/Application Support/AYON`

## Running from source

```
poetry run python start.py [args...]
```

Windows dev helper: `tools/ayon_console.bat --use-dev ...`

## Uploading a custom launcher build

```
./tools/manage.ps1 upload --server <url> --api-key <key>   # Windows
./tools/make.sh  upload --server <url> --api-key <key>     # Linux/macOS
```

The server then distributes the new version to any bundle pinned to it.

---

## Bundles

Declared server-side (`/bundles` API or the web UI). A bundle pins:

- launcher version
- Python dependency package version
- `{addon_name → version}` map
- flag: `isProduction` / `isStaging` / `isDev`
- (dev bundles only) assigned user + optional
  `addonDevelopment` map of `addon_name → local_path`

Launcher resolves:
- default → production bundle
- `--use-staging` → staging bundle
- `--use-dev` → dev bundle belonging to the logged-in user

## Dependency packages

`ayon-dependencies-tool` builds the **dep package** — a zip of wheels for
`ayon-core` + every addon's `client/pyproject.toml`. Each bundle pins a
dep package so all workstations run identical libs.

Bump flow:
1. Merge the addon PR.
2. Rebuild the dep package.
3. Create / update a bundle with the new dep package + addon version.

---

## Developer mode

Minimums (from docs): server ≥ 0.5.0, launcher ≥ 1.0.0-beta.6,
OpenPype addon ≥ 3.17.3.

### Steps

1. Admin marks the user as **Developer** in the Web UI.
2. User refreshes → "Developer mode" checkbox top-right; enable it.
3. Create a dev bundle on the Bundles page, assign yourself, optionally
   map addons to **local paths**.
4. Launch with `ayon --use-dev` (or `AYON_USE_DEV=1` + `AYON_BUNDLE_NAME`).
5. Edit local code → restart launcher → changes apply.

**Gotcha**: local paths only affect **client** code. Server-side changes
still require re-uploading the addon (and usually bumping its version).

---

## Testing

Test layout (`ayon-core` and other addons):

```
tests/
├─ integration/    # end-to-end publish tests per host
├─ unit/           # per-module unit tests
├─ lib/            # helpers (db_handler, file_handler, assert_classes, testing_classes)
└─ resources/      # fixture data
```

Integration tests currently run in **legacy "OpenPype mode"** (MongoDB).
They require `mongorestore`, `mongodump`, `mongoimport` on PATH.

```
./.poetry/bin/poetry run python start.py runtests ./tests/integration/hosts/nuke
```

| Arg | Meaning |
|-----|---------|
| `-m <text>` / `--mark <text>` | pytest mark filter |
| `-p <text>` / `--pyargs <text>` | run by Python package |
| `--test_data_folder <path>` | pre-unzipped fixture dir |
| `-s` / `--persist` | keep DB + files after run |
| `-a <text>` / `--app_variant <text>` | target specific DCC variant |
| `--timeout <secs>` | per-phase timeout |
| `-so` / `--setup_only` | provision DB but skip run |

Hosts with integration tests: Maya, Nuke, AfterEffects, Photoshop.

Each publish test runs: Prepare → Launch → Publish → Validate →
Cleanup (unless `--persist`).

---

## Contributing

- Git flow: `main` (prod), `develop` (PR target), `release/3.x.x`
  (stabilisation).
- Branch naming: `feature/{issue}-{name}`, `bugfix/{issue}-{name}`,
  `hotfix/{issue}-{name}` (hotfix from `main`).
- PRs go to `develop` unless otherwise directed.
- Discuss new features in Ynput Community forum **before**
  implementation.
- Commit-message / lint specifics aren't formally documented — read the
  CI config per repo (many use `ruff` / `black` / `isort`).

---

## Deploying a dev server

`ayon-docker` compose file:

```bash
git clone https://github.com/ynput/ayon-docker
cd ayon-docker
make demo                # macOS/Linux, loads demo_Commercial
./manage.ps1 demo        # Windows
```

Server at `http://localhost:5000`. Admin credentials printed on first
boot.

## Deeper reading

- `11-launcher-dev.md` — full file with Sources

## Sources

- <https://docs.ayon.dev/docs/dev_launcher>
- <https://docs.ayon.dev/docs/dev_dev_mode>
- <https://docs.ayon.dev/docs/dev_requirements>
- <https://docs.ayon.dev/docs/dev_testing>
- <https://docs.ayon.dev/docs/dev_contribute>
- <https://github.com/ynput/ayon-launcher>
- <https://github.com/ynput/ayon-docker>
- <https://github.com/ynput/ayon-dependencies-tool>
