# 22 — Foundry Nuke in Ayon

Repo: `github.com/ynput/ayon-nuke` (100% Python). Nuke is Ayon's primary
compositing DCC; paired with Hiero for editorial via `ayon-hiero`.

## Supported versions & licences

- Nuke **14.0+**
- **Commercial licence only**. Non-commercial and Indie lack the Python
  API surface Ayon needs and will not integrate.

## AYON menu

Installed via `menu.py` on Nuke startup. Groups:

**Global tools**: Creator/Publisher · Loader · Scene Inventory · Workfiles

**Workfile tools**:
- **Set Resolution** — from folder attribs
- **Set Frame Ranges** — from folder attribs
- **Set Colorspace** — per-project from `imageio` settings
- **Apply All Settings** — runs the three above in one action
- **Push to Project** — transfers workfile to another project and
  relocates resources
- **Build Workfile** — instantiate a script from a template

## Product types

| Type | Scene object | Notes |
|------|--------------|-------|
| **Render** | Group containing an AYON-managed Write node | Versionless until published |
| **Prerender** | Same scheme as Render | Intermediate renders that feed later stages |
| **Image** / **Plate** | Loaded via the Loader, not created | Source footage |
| **Review** | Compressed-video output | ProRes 4444 → H.264, etc. |
| **Gizmo** | Custom Nuke gizmo package | Shareable |
| **Nukenodes** | Node collection | Reusable script fragments |

## The versionless write convention

> "Your render file names have no version number until the render is
> fully finished and published."

Artists re-render portions without overwriting complete versions
prematurely. The final version number is only stamped at integrate
time.

## Publishing stages in Nuke

Standard CCVEI, with one extra named phase:

1. **Collect** — scan the script; read instance metadata from write-node
   knobs.
2. **Validate** — checks (frame range, colorspace, resolution, output
   path hygiene).
3. **Extract** — run thumbnails, review transcodes, burnins.
4. **Integrate** — copy files to published locations; update DB;
   notify project trackers (ftrack / SG / Kitsu if configured).
5. **Finalize** — version increment, cleanup staging.

Publisher UX:
- Artist notes box — supervisor communication.
- Per-product **toggle** mid-publish.

## Farm integration

Options on a Render instance:

- **Local** — render here, publish here.
- **Use existing frames** — render was done earlier; publish from the
  existing frames on disk.
- **On Farm** — submit to Deadline (render job + Ayon publish job).
- **Workfile as Deadline dependency** — hold the job until the workfile
  publishes, so you can hit Publish in Nuke without waiting. The
  dependency chain prevents premature execution.

See `24-deadline.md` / skill `ayon-deadline`.

## Tutorials (from `help.ayon.app`)

The "Nuke Workflow Tutorials" article covers four admin-ish topics:

1. **Launcher default mode** — open last version vs. empty on startup.
2. **Workfile naming** — set default via Project Anatomy
   (`workfile_template_profiles`).
3. **Nuke colour management with AYON** — configure via the **imageio**
   addon settings; Nuke picks up OCIO from env or from the host API.
4. **Templated Workfile Builder**:
   - Templates contain `PLACEHOLDER` nodes of two kinds:
     - **Create placeholder** — will be replaced by a newly-created
       publish instance
     - **Load placeholder** — will be replaced by a loaded published
       representation
   - Run **Build Workfile** → Ayon swaps placeholders for real nodes.

The tutorials page does **not** walk through creating specific
Render/Prerender/Review instances step-by-step. For that pattern see
`help.ayon.app/en/articles/4670584-working-with-nuke-in-ayon`.

## Colour management

- Global OCIO config comes from the **imageio** addon settings.
- The host can be launched with `OCIO` env var preset from a launch
  hook, or set via Nuke's Python API at startup.
- Representations carry `colorspaceData` at publish time — see
  `16-site-sync-colorspace.md` / skill `ayon-site-sync-colorspace`.

## Addon layout

```
ayon-nuke/
└─ client/ayon_nuke/
   ├─ api/            # NukeHost, write-node management, gizmo, workfile builder
   ├─ plugins/
   │  ├─ create/      # CreateRender, CreatePrerender, CreateReview, ...
   │  ├─ load/        # LoadRender, LoadImage, LoadGizmo, LoadNukenodes, ...
   │  ├─ publish/     # Collectors / validators / extractors
   │  └─ inventory/
   ├─ hooks/          # PreLaunchHook — NUKE_PATH, env setup
   └─ startup/        # menu.py — installs host + menu
```

Browse `github.com/ynput/ayon-nuke/client/ayon_nuke/` for the current
plugin set.

## Tips / traps

- **Commercial licence only** — test machines on Personal Learning or
  Indie will fail host bootstrap.
- **Write nodes are Groups in disguise** — the artist sees a single
  Write, but Ayon wraps it to hold knobs for instance metadata.
- **Colorspace mismatches** at publish are one of the most common
  validator failures. Confirm the project OCIO is what the Write node
  expects.
- **Workfile builder placeholders** survive across Nuke versions —
  useful templating tool even for advanced users.

## Deeper reading

- `help.ayon.app/en/articles/4670584-working-with-nuke-in-ayon`
- `help.ayon.app/en/articles/2833967-nuke-addon-settings`
- `help.ayon.app/en/articles/4097838-nuke-workflow-tutorials`
- Paired: `ayon-hiero` for editorial conform.

## Sources

- <https://help.ayon.app/en/articles/4670584-working-with-nuke-in-ayon>
- <https://help.ayon.app/en/articles/4097838-nuke-workflow-tutorials>
- <https://github.com/ynput/ayon-nuke>
- <https://github.com/ynput/ayon-nuke/blob/develop/client/ayon_nuke/api/pipeline.py> — `NukeHost(HostBase, IWorkfileHost, ILoadHost, IPublishHost)`
- <https://github.com/ynput/ayon-nuke/blob/develop/client/ayon_nuke/api/lib.py> — `create_write_node()`, `add_write_node()`, `update_write_node_knobs()` (versionless-filename convention via `fpath_template`)
- <https://github.com/ynput/ayon-nuke/blob/develop/client/ayon_nuke/api/workfile_template_builder.py> — `NukeTemplateBuilder`, `NukePlaceholderPlugin`
- <https://github.com/ynput/ayon-nuke/tree/develop/client/ayon_nuke/plugins/workfile_build> — `NukePlaceholderCreatePlugin`, `NukePlaceholderLoadPlugin`
- <https://github.com/ynput/ayon-nuke/tree/develop/client/ayon_nuke/plugins/create> — `CreateWriteRender`, `CreateWritePrerender`, `CreateWriteImage`, `NukeWorkfileCreator`, `CreateCamera`, `CreateGizmo`, `CreateModel`, `CreateSource`, `CreateBackdrop`
- <https://github.com/ynput/ayon-nuke/tree/develop/client/ayon_nuke/plugins/load>
- <https://github.com/ynput/ayon-nuke/blob/develop/server/settings/main.py> — settings (general, imageio, dirmap, scriptsmenu, gizmo, create, publish, load, workfile_builder, templated_workfile_build)
