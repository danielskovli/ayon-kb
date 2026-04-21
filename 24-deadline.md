# 24 — Thinkbox Deadline in Ayon

Repo: `github.com/ynput/ayon-deadline`. Reference render-manager
integration. Source-confirmed 2026-04-21 against develop branch.

## Architecture — two-job pattern

Every farm submission produces **at least two Deadline jobs**:

```
[ DCC Render Job ] → [ AYON Publish Job ]
```

1. **Render Job** — runs the actual render in the target DCC.
2. **AYON Publish Job** — after the render succeeds, re-invokes `ayon`
   on a farm worker to run the Ayon publish pipeline (Collect →
   Integrate) against the rendered output.

Expected files are passed via `instance.data["expectedFiles"]`;
manifests are written by `create_metadata_path()` inside
`ProcessSubmittedJobOnFarm` and consumed by the publish job.

### Houdini split-render variant

Per render instance, the Houdini submitter can emit **two jobs**
(export + render) when `splitRender=True`:

```
[ Houdini Export Job ] → [ Render Job ]     (from HoudiniSubmitDeadline)
                      ↓
           [ AYON Publish Job ]              (from ProcessSubmittedJobOnFarm)
```

- **Export** — Houdini generates intermediate files (`.ass` for Arnold,
  `.ifd` for Mantra). **Only this job consumes a Houdini licence.**
- **Render** — the native renderer's Deadline plugin (Arnold, Mantra)
  processes intermediates. `use_dcc_plugin=False` on this job.
- **Publish** — standard Ayon publish, created by a separate
  `ProcessSubmittedJobOnFarm` submission.

The "three jobs" framing is artist-visible; at the submitter level it's
**two pairs of submissions** (`HoudiniSubmitDeadline` + the generic
`ProcessSubmittedJobOnFarm`).

## Submission is a web-service call

Ayon uses Deadline's **web service** API (`POST /api/jobs`) — not
JobScript shell-outs. `AbstractSubmitDeadline.submit()` handles the
HTTP call; one endpoint; uniform across DCCs.

## Abstract base class

```
client/ayon_deadline/abstract_submit_deadline.py
```

`AbstractSubmitDeadline(pyblish.api.InstancePlugin,
                        AYONPyblishPluginMixin,
                        AbstractMetaInstancePlugin)`

**Extension points** (override in each submitter):

- `get_job_info(job_info, **kwargs)` — host/plugin-specific job config
- `get_plugin_info(**kwargs)` — host/plugin-specific plugin config

**Inherited / concrete helpers**:

- `process(instance)` — pyblish entry; handles single + split submissions
- `get_generic_job_info(instance)` — populates JobInfo name, batch, user,
  frames, env vars
- `get_aux_files()` — auxiliary files (default `[]`)
- `from_published_scene()` — resolves published scene path
- `assemble_payload()` — merges JobInfo + PluginInfo + AuxFiles
- `submit(payload, auth, verify)` — POSTs to Deadline
- `_set_scene_path()`, `_append_job_output_paths()` — output handling

## Submitter plugin inventory (source-confirmed)

Under `client/ayon_deadline/plugins/publish/`:

| File | Class | Base | Order | Hosts | Families |
|------|-------|------|-------|-------|----------|
| `maya/submit_maya_deadline.py` | **`MayaSubmitDeadline`** | `AbstractSubmitDeadline` | `IntegratorOrder + 0.1` | `["maya"]` | `["renderlayer"]` |
| `maya/submit_maya_cache_deadline.py` | **`MayaCacheSubmitDeadline`** | `AbstractSubmitDeadline` | `IntegratorOrder` | `["maya"]` | `["remote_publish_on_farm"]` |
| `nuke/submit_nuke_deadline.py` | **`NukeSubmitDeadline`** | `AbstractSubmitDeadline` | `IntegratorOrder + 0.1` | `["nuke"]` | `["render", "prerender"]` |
| `houdini/submit_houdini_render_deadline.py` | **`HoudiniSubmitDeadline`** | `AbstractSubmitDeadline` | `IntegratorOrder` | `["houdini"]` | `["redshift_rop", "arnold_rop", "mantra_rop", "karma_rop", "vray_rop"]` |
| `houdini/submit_houdini_render_deadline.py` | **`HoudiniSubmitDeadlineUsdRender`** | `HoudiniSubmitDeadline` | inherited | inherited | `["usdrender"]` |
| `houdini/submit_houdini_cache_deadline.py` | **`HoudiniCacheSubmitDeadline`** | `AbstractSubmitDeadline` | `IntegratorOrder` | `["houdini"]` | `["publish.hou"]` |
| `global/submit_publish_job.py` | **`ProcessSubmittedJobOnFarm`** | `pyblish.api.InstancePlugin` | `IntegratorOrder + 0.2` | — | `["deadline.submit.publish.job"]` |
| `houdini/submit_publish_cache_job.py` | **`ProcessSubmittedCacheJobOnFarm`** | `pyblish.api.InstancePlugin` | `IntegratorOrder + 0.2` | `["houdini"]` | `["publish.hou"]` |

Single `NukeSubmitDeadline` handles both `render` and `prerender`
families (not split into two classes as older KB versions guessed).

Additional plugins exist under `aftereffects/`, `blender/`, `cinema4d/`,
`fusion/`, `harmony/`, `max/`, `unreal/` — same pattern.

## Server-side class

`Deadline(BaseServerAddon)` in `server/__init__.py`. Settings model
`DeadlineSettings`; site settings model `DeadlineSiteSettings`.

## Client-side class

```python
class DeadlineAddon(AYONAddon, IPluginPaths):
    # methods:
    #   get_publish_plugin_paths()    — returns host-specific paths
    #   get_server_info_by_name(name)
    #   get_job_info()
    #   submit_job()
```

(`client/ayon_deadline/addon.py`)

## Admin config

### Web-service URL

`ayon+settings://deadline/deadline_urls` — `list[ServerItemSubmodel]`.
Each server has:

| Field | Type | Purpose |
|-------|------|---------|
| `name` | `str` | Unique ID |
| `value` | `str` | URL (e.g. `http://127.0.0.1:8081`) |
| `require_authentication` | `bool` | Gate webservice auth |
| `not_verify_ssl` | `bool` | Skip SSL verify |
| `default_username` | `str` | Default auth user |
| `default_password` | `str` | Default auth password |

**Per-project override**: `ayon+settings://deadline/deadline_server?project=YOUR_PROJECT`
picks one of the configured server names.

**Not confirmed in source**: the multi-server `{server_url}@{token}`
URL format some KB versions referenced. Credentials are per-server via
the fields above; if a legacy `@token` grammar exists, it's not in the
main settings schema. See `99-open-questions.md`.

### Per-site credential overrides

`server/settings/site_settings.py` — `DeadlineSiteSettings.local_settings: list[CredentialPerServerModel]`
lets each workstation/site store its own username/password per server
name.

### Repository plugin install

Deadline-side plugins + events ship in the addon at:

```
client/ayon_deadline/repository/custom/plugins/
├─ GlobalJobPreLoad.py       # event hook (pre-load)
├─ Ayon/
│  ├─ Ayon.py                # AyonDeadlinePlugin(DeadlinePlugin)
│  ├─ Ayon.param             # plugin parameters (AyonExecutable etc.)
│  ├─ Ayon.options           # UI options
│  └─ Ayon.ico
├─ CelAction/
├─ HarmonyAYON/
└─ UnrealEngine5/
```

Copy the contents of `custom/plugins/` into your Deadline repository's
`custom/plugins/` directory (or equivalent mount).

**`GlobalJobPreLoad.py`** runs before every job and injects the Ayon
env (`AYON_SERVER_URL`, `AYON_API_KEY`, `AYON_BUNDLE_NAME`) plus job
metadata. Key functions: `inject_ayon_environment()`,
`inject_openpype_environment()`, `inject_render_job_id()`,
`handle_credentials()`.

**`AyonDeadlinePlugin(DeadlinePlugin)`** in `Ayon/Ayon.py` is the
Deadline-side render-plugin equivalent that invokes `ayon_console publish`
with the metadata JSON.

### Ayon executable path

Configured via `Ayon.param` on the Deadline side; each worker needs
the path to `ayon` / `ayon.exe`.

### Server credentials

The AYON plugin on Deadline reads server URL + API key from a
**service account**, injected by `GlobalJobPreLoad`.

### `AYON_SITE_ID`

Not a settings field — read from the worker's environment by
`client/ayon_deadline/plugins/publish/global/collect_environment_file_to_delete.py`
via `os.environ.get("AYON_SITE_ID")` and folded into the hash key for
the persistent environment file that `GlobalJobPreLoad` caches. Set
`AYON_SITE_ID` uniformly across farm machines so workers share a
single cached env bundle; leave it unset and every worker falls back
to user-only cache keys.

### Optional webservice auth

Username/password per server (see settings table above); admins can set
globally or rely on per-site overrides in `local_settings`.

## Settings schema (source-confirmed)

`server/settings/publish_plugins.py` → `PublishPluginsModel` contains
per-plugin submodels:

| Settings model | Controls |
|----------------|----------|
| `CollectJobInfoModel` | Render job profiles — priority, primary/secondary pool, group, chunk_size, machine_limit, machine_list, department, frame delays |
| `ProcessSubmittedJobOnFarmModel` | Publish job — priority, pool, group, department, AOV filters (`aov_filter: list[AOVFilterSubmodel]`), file skips, family transfers |
| `HoudiniSubmitDeadlineModel` | Export job priority/chunk/group/limits |
| `ProcessCacheJobFarmModel` | Cache publish job settings |
| `MayaSubmitDeadlineModel` | Tile priority, assembler plugin, scene patches, strict error checking |
| `NukeSubmitDeadlineModel` | GPU, continue-on-error, node-class limit groups |
| `FusionSubmitDeadlineModel` | Plugin selection |
| `CollectAYONServerToFarmJobModel` | Enable/disable `AYON_SERVER_URL` forwarding |
| `ValidateExpectedFilesModel` | Enable, active, frame-override allow, family filters |

## Render Job Settings Profiles

Admin knobs exposed to artists at submission time (from
`CollectJobInfoModel`):

- Priority
- Worker groups
- Resource pools (primary + secondary)
- Frame ranges
- Chunk size, machine limit, machine list

When groups/pools are **exposed**, the publisher queries Deadline live
(dropdown of real values, not studio-guessed strings).

## Custom frame rendering

- **Task Frame Range** — scene / folder attrib range (default)
- **Custom Frames Only** — artist-specified (Deadline syntax e.g. `1001-1100x5`)
- **Reuse from Last Version** — `job_info.reuse_last_version` — previous
  version's frames + artist's additions (surgical retakes)

Task vs Custom modes are enforced per submitter / UI, not as a single
enum in settings.

## Dependencies

- Publish job (`ProcessSubmittedJobOnFarm`) always depends on the
  render job.
- Ayon's publishing is **independent per instance** — cross-instance
  ordering falls to Deadline priority + dependencies.
- Some DCCs (Houdini) support multi-job dependency chains natively.

## Tile rendering

`MayaSubmitDeadline._tile_render()` creates:

1. Per-frame tile jobs
2. Assembly jobs (Draft Tile Assembler)
3. AYON publish job

## Workfile-as-dependency (Nuke)

`NukeSubmitDeadline.process()` wires the workfile publish as a Deadline
dependency when `job_info.use_published` is active — lets artists
Publish + Submit in one click without racing the workfile saver.

## Submission flow

1. Artist saves workfile.
2. Create Render instance; configure knobs.
3. (Optional) Publisher dry-run.
4. Publish with target = **On Farm**.
5. DCC-specific submitter (`MayaSubmitDeadline` / `NukeSubmitDeadline` / …)
   builds JobInfo + PluginInfo + AuxFiles, POSTs to Deadline.
6. Separate `ProcessSubmittedJobOnFarm` instance submits the dependent
   publish job pointing at the expected-files manifest.
7. Deadline schedules → workers pick up → `GlobalJobPreLoad` injects env →
   render completes → publish job fires → `ayon_console publish` runs CCVEI.

## Tips / traps

- **`GlobalJobPreLoad` is magic** — if a farm job can't reach the Ayon
  server, check first that the script is installed in the Deadline
  repository and that `AYON_SERVER_URL` / `AYON_API_KEY` arrive in the
  job env.
- **Service account vs user key** — always use a **service** API key
  for farm jobs.
- **Repository plugins must be refreshed** every time the addon
  bumps — contents of `repository/custom/plugins/` change. This is the
  #1 root cause of "farm worked yesterday."
- **Houdini split-render** saves Houdini licences (only the export
  stage consumes one). Not a default; enable per submission.
- **Tile rendering** — multiplies Deadline scheduling overhead; only
  when frame sizes genuinely need it.

## Developer doc status

`docs.ayon.dev/docs/dev_deadline` is thin (test-mode symlinks, version
compat). This file is the authoritative reference; it's source-verified
against the current `ynput/ayon-deadline` develop branch.

## Deeper reading

- `help.ayon.app/en/articles/8138091-working-with-deadline-in-ayon`
- `help.ayon.app/en/articles/5372986-configure-deadline-addon`
- `help.ayon.app/en/articles/6879829-configure-royal-render-addon` — analogous for Royal Render
- Related: `22-nuke.md` (Nuke farm), `20-maya.md` (Maya farm),
  `21-houdini.md` (Houdini split render).

## Sources

- <https://help.ayon.app/en/articles/8138091-working-with-deadline-in-ayon>
- <https://help.ayon.app/en/articles/5372986-configure-deadline-addon>
- <https://docs.ayon.dev/docs/dev_deadline>
- <https://github.com/ynput/ayon-deadline>
- <https://github.com/ynput/ayon-deadline/blob/develop/client/ayon_deadline/abstract_submit_deadline.py> (AbstractSubmitDeadline + extension points)
- <https://github.com/ynput/ayon-deadline/blob/develop/client/ayon_deadline/addon.py> (DeadlineAddon)
- <https://github.com/ynput/ayon-deadline/tree/develop/client/ayon_deadline/plugins/publish> (all submitter classes)
- <https://github.com/ynput/ayon-deadline/blob/develop/server/settings/main.py> (settings)
- <https://github.com/ynput/ayon-deadline/blob/develop/server/settings/publish_plugins.py> (per-plugin settings)
- <https://github.com/ynput/ayon-deadline/blob/develop/client/ayon_deadline/repository/custom/plugins/GlobalJobPreLoad.py>
- <https://github.com/ynput/ayon-deadline/blob/develop/client/ayon_deadline/repository/custom/plugins/Ayon/Ayon.py>
