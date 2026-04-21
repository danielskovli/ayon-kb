---
name: ayon-nuke
description: Foundry Nuke in Ayon (`ayon-nuke`, Nuke 14+, Nuke commercial licence only — non-commercial / Indie unsupported). Paired with `ayon-hiero`. Addon menu: Set Resolution / Set Frame Range / Set Colorspace / Apply All / Push to Project / Build Workfile. Product types: Render, Prerender, Image/Plate, Review, Nuke Gizmo, Nukenodes. AYON-managed Write node convention (versionless render Nuke — paths contain no version until publish). CCVEI publish stages Collect→Validate→Extract→Integrate→Finalize. Farm targets: Local / Use existing frames / On Farm / workfile dependency Deadline. Templated Workfile Builder with Create/Load `PLACEHOLDER` nodes. Nuke colour management via `imageio` addon. Use when writing Nuke creators/loaders, debugging publish, helping an artist, or configuring Nuke templates.
---

# Foundry Nuke in Ayon

Repo: `github.com/ynput/ayon-nuke`. Paired with `ayon-hiero` for
editorial conform.

## Versions & licences

- **Nuke 14.0+**.
- **Commercial licence only**. Non-commercial + Indie lack Python API
  surface → host bootstrap fails.

## AYON menu

Global tools: Creator/Publisher · Loader · Scene Inventory · Workfiles.

Workfile tools:
- **Set Resolution** — from folder attribs
- **Set Frame Ranges** — from folder attribs
- **Set Colorspace** — from `imageio` settings
- **Apply All Settings** — runs the three above
- **Push to Project** — transfer script + resources to another project
- **Build Workfile** — instantiate from template

## Product types

| Type | Scene object | Notes |
|------|--------------|-------|
| **Render** | Group wrapping an AYON-managed Write node | Versionless until published |
| **Prerender** | Same scheme | Intermediate renders feeding later stages |
| **Image / Plate** | Loaded via Loader | Source footage |
| **Review** | Compressed video | ProRes 4444 → H.264 etc. |
| **Gizmo** | Nuke gizmo package | Shareable |
| **Nukenodes** | Node collection | Reusable script fragment |

## Versionless Write convention

> Render file names **have no version number** until the render is
> fully finished and published.

Artists re-render portions without overwriting completed versions
prematurely; the final version stamps only at integrate.

## Publishing stages (Nuke's extra phase)

1. Collect — read instance metadata from Write-node knobs
2. Validate
3. Extract — thumbnails, reviews, burnins
4. Integrate — copy, update DB, notify trackers
5. **Finalize** — version increment, cleanup staging

Publisher UX: artist notes, per-product toggle mid-publish.

## Farm integration

Render instance knob options:
- **Local** — render + publish here
- **Use existing frames** — publish from frames on disk
- **On Farm** — submit to Deadline (render job + publish job)
- **Workfile as Deadline dependency** — hold render until workfile
  publishes, so Publish+Submit in one click doesn't race the workfile
  save

See `ayon-deadline`.

## Tutorials

From the Nuke Workflow Tutorials article:

1. **Launcher default mode** — open last vs empty on startup.
2. **Workfile naming** — set via Project Anatomy's
   `workfile_template_profiles`.
3. **Nuke colour management** — configure through the **imageio**
   addon; Nuke picks up OCIO from env or host API at startup.
4. **Templated Workfile Builder**:
   - Templates include **`PLACEHOLDER`** nodes of two types:
     - **Create placeholder** — replaced by a new publish instance
     - **Load placeholder** — replaced by a loaded published rep
   - Run **Build Workfile** → Ayon swaps them for real nodes.

The tutorials page does **not** walk through creating specific
Render/Prerender/Review instances step-by-step — for that see the
Working-with-Nuke article.

## Addon layout

```
ayon-nuke/client/ayon_nuke/
├─ api/            # NukeHost, write-node management, gizmo, workfile builder
├─ plugins/
│  ├─ create/      # CreateRender, CreatePrerender, CreateReview, ...
│  ├─ load/        # LoadRender, LoadImage, LoadGizmo, LoadNukenodes, ...
│  ├─ publish/     # Collectors / validators / extractors
│  └─ inventory/
├─ hooks/          # PreLaunchHook — NUKE_PATH, env setup
└─ startup/        # menu.py — installs host + menu
```

## Tips / traps

- Test machines on Indie / Non-commercial licences will fail; use
  commercial.
- Write nodes are **Groups** in disguise — the Ayon Write wraps real
  knobs with metadata knobs.
- Colorspace mismatches at publish are the single most common
  validator failure. Confirm the project OCIO matches what the Write
  node expects.
- Workfile builder placeholders survive across Nuke versions — good
  templating tool even for senior users.

## Deeper reading

- `22-nuke.md` — full file with Sources
- `help.ayon.app/en/articles/2833967-nuke-addon-settings`
- Skill `ayon-deadline` — farm submission
- Skill `ayon-site-sync-colorspace` — OCIO + representation metadata

## Sources

- <https://help.ayon.app/en/articles/4670584-working-with-nuke-in-ayon>
- <https://help.ayon.app/en/articles/4097838-nuke-workflow-tutorials>

### Source repo (verified 2026-04-21 against develop)

- <https://github.com/ynput/ayon-nuke> — addon root
- <https://github.com/ynput/ayon-nuke/blob/develop/client/ayon_nuke/api/pipeline.py> — `NukeHost(HostBase, IWorkfileHost, ILoadHost, IPublishHost)`
- <https://github.com/ynput/ayon-nuke/blob/develop/client/ayon_nuke/api/lib.py> — `create_write_node()`, `add_write_node()`, `update_write_node_knobs()` (versionless-filename convention at `fpath_template`)
- <https://github.com/ynput/ayon-nuke/blob/develop/client/ayon_nuke/api/workfile_template_builder.py> — `NukeTemplateBuilder`, `NukePlaceholderPlugin`
- <https://github.com/ynput/ayon-nuke/tree/develop/client/ayon_nuke/plugins/workfile_build> — `NukePlaceholderCreatePlugin`, `NukePlaceholderLoadPlugin`
- <https://github.com/ynput/ayon-nuke/tree/develop/client/ayon_nuke/plugins/create> — `CreateWriteRender`, `CreateWritePrerender`, `CreateWriteImage` (all extend `NukeWriteCreator`), plus `CreateCamera`, `CreateGizmo`, `CreateModel`, `CreateSource`, `CreateBackdrop`, `NukeWorkfileCreator`
- <https://github.com/ynput/ayon-nuke/tree/develop/client/ayon_nuke/plugins/load> — `LoadClip`, `LoadImage`, `LoadGizmo`, `LoadCamera`, `LoadCameraUSD`, `LoadUSD`, `LoadEffects`, `LoadMatchmove`, `LoadModel`, `LoadOciolook`, `LoadScriptPrecomp`, `LoadBackdrop`
- <https://github.com/ynput/ayon-nuke/blob/develop/server/settings/main.py> — settings sections (general, imageio, dirmap, scriptsmenu, gizmo, create, publish, load, workfile_builder, templated_workfile_build)
