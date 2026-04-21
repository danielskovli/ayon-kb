---
name: ayon-dcc-integrations
description: Ayon per-DCC integration map — which addon lives at which repo. Use when asked "which repo", "where's the Maya addon", "is there a Substance integration", "DCC addon repo", when porting an integration, or finding artist/admin help articles per DCC. Covers DCC integrations: `ayon-maya`, `ayon-blender`, `ayon-houdini`, `ayon-nuke`, `ayon-hiero`, `ayon-fusion`, `ayon-3dsmax`, `ayon-unreal`, `ayon-aftereffects`, `ayon-photoshop`, `ayon-premiere`, `ayon-tvpaint`, `ayon-resolve`, `ayon-cinema4d`, `ayon-flame`, `ayon-substance`, `ayon-zbrush`, `ayon-mocha`. Plus production trackers and farm: `ayon-ftrack`, `ayon-shotgrid`, `ayon-kitsu`, `ayon-deadline`, Slack, Jira, Perforce.
---

# Ayon DCC & service integrations

Each supported DCC is a separate repo at `github.com/ynput/ayon-<host>`.
Structure follows the addon pattern (see `ayon-addon-development`) plus
the host pattern (see `ayon-host-integration`).

## DCC integrations (maintained as of 2026-04)

| Host | Repo | Notes |
|------|------|-------|
| Maya | `ayon-maya` | Feature-complete reference |
| Blender | `ayon-blender` | Pure-Python, clean reference |
| Houdini | `ayon-houdini` | Includes USD plumbing |
| Nuke | `ayon-nuke` | Paired with `ayon-hiero` |
| Hiero | `ayon-hiero` | |
| Fusion | `ayon-fusion` | |
| 3ds Max | `ayon-3dsmax` | |
| Unreal | `ayon-unreal` | UE Python API |
| AfterEffects | `ayon-aftereffects` | |
| Photoshop | `ayon-photoshop` | |
| Premiere | `ayon-premiere` | |
| TVPaint | `ayon-tvpaint` | |
| Resolve (Studio) | `ayon-resolve` | |
| Cinema 4D | `ayon-cinema4d` | |
| Flame | `ayon-flame` | |
| Harmony | `ayon-harmony` | |
| Motionbuilder | `ayon-motionbuilder` | |
| Substance Painter / Designer | `ayon-substancepainter`, `ayon-substancedesigner` | |
| Zbrush | `ayon-zbrush` | |
| Mocha Pro | `ayon-mocha` | |
| Silhouette | `ayon-silhouette` | |
| SpeedTree | `ayon-speedtree` | |
| Marvelous Designer | `ayon-marvelousdesigner` | |
| Wrap (Faceform) | `ayon-wrap` | |
| CelAction | `ayon-celaction` | |
| 3DEqualizer | `ayon-3dequalizer` | |
| OpenRV | `ayon-openrv` | |
| DJV | `ayon-djv` | |
| TrayPublisher | `ayon-traypublisher` | DCC-less; plates / refs / delivery |
| USD | `ayon-usd` | Resolver + pipeline |

## Production / review / farm integrations

| Target | Repo | Role |
|--------|------|------|
| ftrack | `ayon-ftrack` | Project/folder/task sync (service) |
| Shotgrid | `ayon-shotgrid` | SG sync (service) |
| Kitsu | `ayon-kitsu` | Kitsu sync (service) |
| Jira | `ayon-jira` | Notifications / ticket sync |
| Slack | `ayon-slack` | Notifications on events |
| Deadline | `ayon-deadline` | Render-farm submit + polling |
| Royal Render | settings via Core | Render-farm submit |
| Perforce | `ayon-perforce` | P4 version control for workfiles |

## Conventions inside a DCC addon

- `client/ayon_<host>/api/` — all `import <dcc>` calls stay here; keep
  module-top-level imports lazy.
- `client/ayon_<host>/startup/` — whatever the DCC runs at boot
  (`userSetup.py` for Maya, `menu.py` for Nuke, etc.).
- `client/ayon_<host>/hooks/` — `PreLaunchHook` / `PostLaunchHook`.
- `client/ayon_<host>/plugins/{create,load,publish,inventory}/` —
  host-scoped plugins (filter via class attr `hosts = ["<host>"]`).
- Depends on `ayon-applications` (launch env) and `ayon-core` (pipeline
  glue).

## `ayon-applications`

Central addon that defines `Application` and `ApplicationManager`. Used by
the launcher to:

- list available apps per project
- compose env vars (with per-host hooks)
- spawn the DCC subprocess
- handle workfile templating

For a brand-new DCC you either extend `ayon-applications` or define an
app group in its settings.

## Adding a new host — short checklist

See `ayon-host-integration` for the full playbook.

1. Scaffold an addon.
2. Implement `HostBase` + `IWorkfileHost` + `ILoadHost` + `IPublishHost`.
3. Write at least one `PreLaunchHook` (env + Python path).
4. Bootstrap under `startup/` — `install_host(YourHost())` + menu wiring.
5. Ship at minimum: a `Creator` for `workfile`, a workfile collector, one
   `LoaderPlugin` per supported representation format.
6. Declare the app in `ayon-applications` (or extend).
7. Upload + dev bundle + test via `ayon --use-dev`.

## Reading tips

- Features missing in `ayon-core` that work in Maya live in
  `ayon-maya/client/ayon_maya/api/`.
- New representation type? Copy a similar plugin in
  `ayon-core/client/ayon_core/plugins/publish/` and specialise.
- DCC bootstrap differs wildly — always inspect `startup/` of the nearest
  existing host.

## Artist help articles per DCC

Each DCC has a "Working with <X> in AYON" + (usually) a settings article
at `help.ayon.app/en/collections/1925761-dcc-integrations`. See
`13-resources.md` in the KB root for the full URL index.

## Deeper reading

- `12-dcc-integrations.md` — full file with Sources
- Skill `ayon-host-integration` — implementation patterns

## Sources

- <https://github.com/orgs/ynput/repositories?q=ayon-> — live repo list
- <https://help.ayon.app/en/collections/1925761-dcc-integrations>
- <https://docs.ayon.dev/docs/dev_host_implementation>

### DCC source repos

- <https://github.com/ynput/ayon-maya> · <https://github.com/ynput/ayon-blender> · <https://github.com/ynput/ayon-houdini> · <https://github.com/ynput/ayon-nuke> · <https://github.com/ynput/ayon-hiero> · <https://github.com/ynput/ayon-3dsmax> · <https://github.com/ynput/ayon-unreal> · <https://github.com/ynput/ayon-aftereffects> · <https://github.com/ynput/ayon-photoshop> · <https://github.com/ynput/ayon-premiere> · <https://github.com/ynput/ayon-fusion> · <https://github.com/ynput/ayon-tvpaint> · <https://github.com/ynput/ayon-resolve> · <https://github.com/ynput/ayon-cinema4d> · <https://github.com/ynput/ayon-flame> · <https://github.com/ynput/ayon-harmony> · <https://github.com/ynput/ayon-motionbuilder> · <https://github.com/ynput/ayon-substancepainter> · <https://github.com/ynput/ayon-substancedesigner> · <https://github.com/ynput/ayon-zbrush> · <https://github.com/ynput/ayon-mocha> · <https://github.com/ynput/ayon-silhouette> · <https://github.com/ynput/ayon-speedtree> · <https://github.com/ynput/ayon-marvelousdesigner> · <https://github.com/ynput/ayon-wrap> · <https://github.com/ynput/ayon-celaction> · <https://github.com/ynput/ayon-3dequalizer> · <https://github.com/ynput/ayon-openrv> · <https://github.com/ynput/ayon-djv> · <https://github.com/ynput/ayon-traypublisher> · <https://github.com/ynput/ayon-usd>

### Production / farm repos

- <https://github.com/ynput/ayon-ftrack> · <https://github.com/ynput/ayon-shotgrid> · <https://github.com/ynput/ayon-kitsu> · <https://github.com/ynput/ayon-jira> · <https://github.com/ynput/ayon-slack> · <https://github.com/ynput/ayon-deadline> · <https://github.com/ynput/ayon-perforce> · <https://github.com/ynput/ayon-applications>
