# 12 — DCC integrations

Each supported DCC has its own repo at `github.com/ynput/ayon-<host>`.
Structure follows `04-addon-structure.md` + `07-host-integration.md`.

## Currently maintained (as of research date)

| Host | Repo | Notes |
|------|------|-------|
| Maya | `ayon-maya` | Feature-complete reference integration |
| Blender | `ayon-blender` | Pure-Python, clean reference |
| Houdini | `ayon-houdini` | Includes USD plumbing |
| Nuke | `ayon-nuke` | With Hiero via `ayon-hiero` |
| Hiero | `ayon-hiero` | |
| Fusion | `ayon-fusion` | |
| 3ds Max | `ayon-3dsmax` | |
| AfterEffects | `ayon-aftereffects` | |
| Photoshop | `ayon-photoshop` | |
| TVPaint | `ayon-tvpaint` | |
| Unreal | `ayon-unreal` | UE Python API |
| Substance Painter / Designer | `ayon-substance*` | |
| Cinema 4D | `ayon-cinema4d` | |
| Flame | `ayon-flame` | |
| Resolve | `ayon-resolve` | |
| Mocha | `ayon-mocha` | |
| OpenRV | `ayon-openrv` | |
| DJV | `ayon-djv` | |
| TrayPublisher | `ayon-traypublisher` | DCC-less publisher, useful for plates / refs |
| USD | `ayon-usd` | Resolver + pipeline |

## Production / review integrations

| Target | Repo | Role |
|--------|------|------|
| ftrack | `ayon-ftrack` | Project/folder/task sync (service) |
| Shotgrid | `ayon-shotgrid` | Same, for SG |
| Kitsu | `ayon-kitsu` | Same, for Kitsu |
| Jira | `ayon-jira` | Notifications / ticket sync |
| Slack | `ayon-slack` | Notifications on events |
| Deadline | `ayon-deadline` | Render farm submit + polling |
| Perforce | `ayon-perforce` | P4 workfiles |

## DCC integration conventions

- `client/ayon_<host>/api/` — all `import <dcc>` happens inside functions here.
- `client/ayon_<host>/startup/` — whatever the DCC runs automatically on boot
  (`userSetup.py` for Maya, `menu.py` for Nuke, etc.).
- `client/ayon_<host>/hooks/` — `PreLaunchHook` / `PostLaunchHook` classes.
- `client/ayon_<host>/plugins/{create,load,publish,inventory}/` —
  host-specific plugins; filter with class attr `hosts = ["<host>"]`.
- Usually relies on `ayon-applications` (for launching) and `ayon-core`
  (for pipeline glue).

## `ayon-applications`

Central addon that defines `Application` and `ApplicationManager` — used by
the launcher to:

- list available apps per project
- compose env vars (with per-host hooks)
- spawn the DCC subprocess
- handle workfile templating

If you're shipping a brand-new DCC, you'll either extend `ayon-applications`
or define an app group in its settings.

## Adding a new host — short checklist

1. Scaffold an addon (`04-addon-structure.md`).
2. In `client/ayon_<host>/api/`, subclass `HostBase` + `IWorkfileHost` +
   `ILoadHost` + `IPublishHost`. Implement the required methods.
3. Add a hook under `client/ayon_<host>/hooks/` that prepares env vars for
   the DCC and injects Python paths.
4. Bootstrap script under `client/ayon_<host>/startup/` calls
   `install_host(<YourHost>())` and wires the menu.
5. In `client/ayon_<host>/plugins/`, write at least:
   - a `Creator` for `workfile`
   - a `Collector` that collects the current workfile
   - a `LoaderPlugin` for one representation format
6. Declare app in `ayon-applications` settings (or extend).
7. Build + upload + bundle + test via dev mode.

## Reading tips

- If a feature isn't in `ayon-core` but you can see it in Maya, check
  `ayon-maya/client/ayon_maya/api/` — much logic is host-owned.
- Publishing template for a new representation type: copy a similar plugin
  in `ayon-core/client/ayon_core/plugins/publish/` and specialise.
- DCC-facing bootstrap differs wildly between hosts — always inspect the
  `startup/` dir of the nearest existing host.

## Sources

- <https://github.com/orgs/ynput/repositories?q=ayon-> — live repo list for all DCC / service addons
- <https://help.ayon.app/en/collections/1925761-dcc-integrations> — artist-facing "Working with X" + "Configure X Addon" articles per host (see `17-user-workflows.md` for the index)
- <https://docs.ayon.dev/docs/dev_host_implementation> — integration patterns
- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/host> — host base classes + interfaces source
