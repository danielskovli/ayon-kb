---
name: ayon-deadline
description: Thinkbox Deadline integration for Ayon (`ayon-deadline`, farm render, farm publish, farm job failed). Two-job pattern — DCC Render Job → AYON Publish Job via `ProcessSubmittedJobOnFarm`. Custom submitters extend `AbstractSubmitDeadline` (override `get_job_info` / `get_plugin_info`): `MayaSubmitDeadline`, `MayaCacheSubmitDeadline`, `NukeSubmitDeadline`, `HoudiniSubmitDeadline`, `HoudiniSubmitDeadlineUsdRender`, `HoudiniCacheSubmitDeadline`, `ProcessSubmittedCacheJobOnFarm`. Deadline-side `AyonDeadlinePlugin` + `GlobalJobPreLoad` env injection at `client/ayon_deadline/repository/custom/plugins/`. Settings: `deadline_urls` + auth fields, per-project `deadline_server`, per-site `local_settings`, `reuse_last_version`, render pool, render group. Houdini `splitRender=True` export+render split, Maya tile rendering, Nuke workfile-as-dependency. Use when writing a submitter, configuring Deadline, or debugging a farm-side publish failure.
---

# Thinkbox Deadline in Ayon

Repo: `github.com/ynput/ayon-deadline`. Source-confirmed against develop branch.

## Two-job pattern

```
[ DCC Render Job ] → [ AYON Publish Job ]
```

Expected files travel via `instance.data["expectedFiles"]`; the manifest
is written by `create_metadata_path()` inside `ProcessSubmittedJobOnFarm`.

## Houdini split-render variant

Per render instance with `splitRender=True`:

```
[ Houdini Export Job ] → [ Render Job ]   (from HoudiniSubmitDeadline)
                     ↓
          [ AYON Publish Job ]             (from ProcessSubmittedJobOnFarm)
```

- Export consumes a Houdini licence; render uses the native renderer's
  Deadline plugin (`use_dcc_plugin=False`).
- Publish is created by a separate `ProcessSubmittedJobOnFarm` submission.

"Three jobs" is artist-visible; submitter-level it's two pairs of
submissions.

## Abstract base class

`client/ayon_deadline/abstract_submit_deadline.py`:

```python
class AbstractSubmitDeadline(
    pyblish.api.InstancePlugin,
    AYONPyblishPluginMixin,
    AbstractMetaInstancePlugin,
):
    # extension points (override in each DCC submitter):
    def get_job_info(self, job_info, **kwargs): ...
    def get_plugin_info(self, **kwargs): ...

    # concrete helpers:
    #   process(instance) — pyblish entry
    #   get_generic_job_info(instance) — name, batch, user, frames, env
    #   get_aux_files() → list
    #   from_published_scene() — published scene path
    #   assemble_payload() — JobInfo + PluginInfo + AuxFiles
    #   submit(payload, auth, verify) — POST /api/jobs
```

## Submitter plugin inventory (source-confirmed)

Under `client/ayon_deadline/plugins/publish/`:

| File | Class | Parent | Order | Hosts | Families |
|------|-------|--------|-------|-------|----------|
| `maya/submit_maya_deadline.py` | **`MayaSubmitDeadline`** | `AbstractSubmitDeadline` | `IntegratorOrder + 0.1` | `["maya"]` | `["renderlayer"]` |
| `maya/submit_maya_cache_deadline.py` | **`MayaCacheSubmitDeadline`** | `AbstractSubmitDeadline` | `IntegratorOrder` | `["maya"]` | `["remote_publish_on_farm"]` |
| `nuke/submit_nuke_deadline.py` | **`NukeSubmitDeadline`** | `AbstractSubmitDeadline` | `IntegratorOrder + 0.1` | `["nuke"]` | `["render", "prerender"]` |
| `houdini/submit_houdini_render_deadline.py` | **`HoudiniSubmitDeadline`** | `AbstractSubmitDeadline` | `IntegratorOrder` | `["houdini"]` | `["redshift_rop", "arnold_rop", "mantra_rop", "karma_rop", "vray_rop"]` |
| `houdini/submit_houdini_render_deadline.py` | **`HoudiniSubmitDeadlineUsdRender`** | `HoudiniSubmitDeadline` | inherited | inherited | `["usdrender"]` |
| `houdini/submit_houdini_cache_deadline.py` | **`HoudiniCacheSubmitDeadline`** | `AbstractSubmitDeadline` | `IntegratorOrder` | `["houdini"]` | `["publish.hou"]` |
| `global/submit_publish_job.py` | **`ProcessSubmittedJobOnFarm`** | `pyblish.api.InstancePlugin` | `IntegratorOrder + 0.2` | — | `["deadline.submit.publish.job"]` |
| `houdini/submit_publish_cache_job.py` | **`ProcessSubmittedCacheJobOnFarm`** | `pyblish.api.InstancePlugin` | `IntegratorOrder + 0.2` | `["houdini"]` | `["publish.hou"]` |

Extra DCC folders (aftereffects, blender, cinema4d, fusion, harmony,
max, unreal) follow the same pattern.

`NukeSubmitDeadline` handles **both** `render` and `prerender` —
single class, not two.

## Server + client addon classes

```python
# server/__init__.py
class Deadline(BaseServerAddon):
    settings_model = DeadlineSettings
    site_settings_model = DeadlineSiteSettings

# client/ayon_deadline/addon.py
class DeadlineAddon(AYONAddon, IPluginPaths):
    def get_publish_plugin_paths(self): ...
    def get_server_info_by_name(self, name): ...
    def get_job_info(self): ...
    def submit_job(self): ...
```

## Admin config

### `deadline_urls` — list of servers

Each entry is a `ServerItemSubmodel`:

| Field | Type | Purpose |
|-------|------|---------|
| `name` | `str` | Unique ID |
| `value` | `str` | URL |
| `require_authentication` | `bool` | Gate auth |
| `not_verify_ssl` | `bool` | Skip SSL verify |
| `default_username`, `default_password` | `str` | Default creds |

### Per-project server override

`ayon+settings://deadline/deadline_server?project=YOUR_PROJECT` picks
one of the configured server `name`s.

**NOT in source**: the `{server_url}@{token}` multi-server format older
KB versions referenced. Credentials are per-server via the fields above.
Flagged in `99-open-questions.md`.

### Per-site credential overrides

`server/settings/site_settings.py` →
`DeadlineSiteSettings.local_settings: list[CredentialPerServerModel]`
for workstation/site-local auth.

### Deadline repository plugins

Ayon ships them at:

```
client/ayon_deadline/repository/custom/plugins/
├─ GlobalJobPreLoad.py   # Deadline event hook
├─ Ayon/
│  ├─ Ayon.py            # AyonDeadlinePlugin(DeadlinePlugin)
│  ├─ Ayon.param
│  ├─ Ayon.options
│  └─ Ayon.ico
├─ CelAction/
├─ HarmonyAYON/
└─ UnrealEngine5/
```

Copy the contents of `custom/plugins/` into the Deadline repository.

**`GlobalJobPreLoad.py`** runs before every job and injects:
- `AYON_SERVER_URL`, `AYON_API_KEY`, `AYON_BUNDLE_NAME`
- job metadata

Key funcs: `inject_ayon_environment()`, `inject_openpype_environment()`,
`inject_render_job_id()`, `handle_credentials()`.

**`AyonDeadlinePlugin(DeadlinePlugin)`** in `Ayon/Ayon.py` is the
Deadline-side render-plugin that invokes `ayon_console publish` with
the metadata JSON.

### `AYON_SITE_ID`

Not a settings field, but **read from the worker environment** by
`collect_environment_file_to_delete.py` as part of the cache key for
the shared env file that `GlobalJobPreLoad` produces. Set it
uniformly across farm nodes so workers share one cached env bundle;
unset → per-user cache keys.

## Settings models (source-confirmed)

`server/settings/publish_plugins.py` → `PublishPluginsModel`:

| Model | Controls |
|-------|----------|
| `CollectJobInfoModel` | Render job profiles: priority, pools, group, chunk_size, machine_limit, machine_list, dept |
| `ProcessSubmittedJobOnFarmModel` | Publish job: priority, pool, group, `aov_filter`, file skips, family transfers |
| `HoudiniSubmitDeadlineModel` | Export job priority/chunk/group/limits |
| `ProcessCacheJobFarmModel` | Cache publish settings |
| `MayaSubmitDeadlineModel` | Tile priority, assembler plugin, scene patches, strict errors |
| `NukeSubmitDeadlineModel` | GPU, continue-on-error, node-class limit groups |
| `FusionSubmitDeadlineModel` | Plugin selection |
| `CollectAYONServerToFarmJobModel` | Enable/disable `AYON_SERVER_URL` forwarding |
| `ValidateExpectedFilesModel` | Enable, active, frame-override allow, family filters |

## Custom frame modes

- **Task Frame Range** (default)
- **Custom Frames Only** (`1001-1100x5`)
- **Reuse from Last Version** (`job_info.reuse_last_version`) — previous
  version's frames + artist's additions

## Tile rendering (Maya)

`MayaSubmitDeadline._tile_render()`:
1. Per-frame tile jobs
2. Assembly jobs (Draft Tile Assembler)
3. AYON publish job

## Workfile-as-dependency (Nuke)

`NukeSubmitDeadline.process()` uses `job_info.use_published` to wire
the workfile publish as a Deadline dependency. Publish + Submit in one
click without racing the workfile saver.

## Tips / traps

- **Farm can't reach Ayon server?** → `GlobalJobPreLoad` not installed
  or `AYON_SERVER_URL`/`AYON_API_KEY` missing from env.
- **"Worked yesterday" after addon update?** → refresh
  `repository/custom/plugins/` into the Deadline repo. #1 cause.
- **Service-account API key only** on farm. Rotating a user's key
  kills every scheduled job.
- **Houdini split-render** — only enable when licence-tight.
- **Tile rendering** — multiplies scheduling overhead.

## Deeper reading

- `24-deadline.md` — full file with Sources
- Skill `ayon-maya` — Maya render/tile flow
- Skill `ayon-nuke` — Nuke on-farm + workfile dependency
- Skill `ayon-houdini` — Houdini split-export
- Skill `ayon-publishing` — publish pipeline the publish job re-runs

## Sources

- <https://help.ayon.app/en/articles/8138091-working-with-deadline-in-ayon>
- <https://help.ayon.app/en/articles/5372986-configure-deadline-addon>
- <https://github.com/ynput/ayon-deadline/blob/develop/client/ayon_deadline/abstract_submit_deadline.py>
- <https://github.com/ynput/ayon-deadline/blob/develop/client/ayon_deadline/addon.py>
- <https://github.com/ynput/ayon-deadline/tree/develop/client/ayon_deadline/plugins/publish>
- <https://github.com/ynput/ayon-deadline/blob/develop/server/settings/main.py>
- <https://github.com/ynput/ayon-deadline/blob/develop/server/settings/publish_plugins.py>
- <https://github.com/ynput/ayon-deadline/blob/develop/client/ayon_deadline/repository/custom/plugins/GlobalJobPreLoad.py>
- <https://github.com/ynput/ayon-deadline/blob/develop/client/ayon_deadline/repository/custom/plugins/Ayon/Ayon.py>
