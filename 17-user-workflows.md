# 17 — User workflows (artist-facing)

This file covers the *artist* / *TD* side of Ayon — "how do I actually use
the thing." For developer plumbing see `05`–`16`.

## First-day workflow (launching a task)

1. **Install** the launcher — download from the studio's Ayon server, run
   the installer, and on first run enter server URL + credentials.
2. **Tray icon** appears. Color indicates mode (production vs staging).
3. Click tray → **open Launcher window**.
4. **Select**: Project → Folder → Task. Use the top filter to narrow down.
   You can toggle "show only my tasks."
5. **Actions pane** now shows the DCCs / tools configured for this task.
   Click one to launch.
6. The DCC boots with the right env + (by default) your last workfile.
   Right-click the app icon to start empty or pick a specific workfile.

Additional per-context tools:

- **Explore Here** — opens file explorer at the work dir.
- **Show in AYON** — jumps to the context in the web UI.
- **Terminal** — shell with the task's env pre-loaded.

## Workfiles tool

Replaces File → Open / File → Save Asfrom inside each DCC. Available as
`AYON → Workfiles`.

- Open a file: select from the list (newest first) → **Open** or double-click.
- Save: **Save As** → optional sub-version (e.g. `_mattepainting`) → confirm.
  `Ctrl+S` does an increment-save.
- Automatic version bump: `v001` → `v002` → … with manual override available.
- Artist notes can be attached per workfile.
- Publishing may trigger a further automatic bump depending on studio
  settings.

Version numbers are **per task**, tracked by the anatomy's work template.

## Create / Publish

Two phases of the same tool (`AYON → Publisher`).

### Create phase

1. Select content in the scene (a mesh, a camera, a comp tree, a write node…).
2. Open Creator (via AYON menu or Publisher's *Create* tab).
3. Pick a product type — e.g. **Model**, **Camera**, **Render**.
4. Enter a **variant** (commonly `Main`, additional ones like `High`, `Low`,
   `V01`).
5. Click **Create**.

What actually happens in the scene varies per DCC:

- **Maya**: creates an `objectSet` with metadata on custom attrs.
- **Nuke**: adds a Write node with metadata on hidden knobs.
- **Resolve**: timeline markers with metadata in notes.

Every asset is expected to have a `Main` product; additional variants are
optional.

### Publish phase

The Publisher UI shows two panels:

- **Left** — list of instances about to publish.
- **Right** — their create/publish attributes (toggles set by plugins).

Run **dry-run** to execute Collect + Validate without Extract + Integrate.
Validation errors are grouped by title with repair actions where available.
Fix → re-run → click the **play** button to finish Extract + Integrate.

Failed validation shows a report panel. **`PublishXmlValidationError`**
items have per-plugin help text. **`PublishValidationError`** shows the
plugin's own message. **`KnownPublishError`** is unrecoverable; retry later.

## Loader

`AYON → Load…` — bring published products into the current workfile.

- **Filters**: product name (text), product type (checklist).
- **Version** column: double-click to pick a specific version. Hero
  versions show their source version in square brackets, e.g. `[v003]`.
- **Enable Grouping**: `Ctrl+G` assigns a group; groups persist per
  project.
- **Site widget** (when Site Sync is on): per-representation availability
  badge + "download" button.
- Library projects are selectable — loading across projects is supported.

The action actually performed by "Load" depends on which loader the
artist picked for the representation (there may be multiple compatible
ones).

## Scene Inventory

`AYON → Manage → Inventory` — status + lifecycle for every loaded
container.

- **Outdated items** go red — has a newer version than loaded.
- Actions per item / selection:
  - **Update to Latest**
  - **Change to Hero**
  - **Set Version** — explicit version picker
  - **Switch Asset** — load a different product / representation
  - **Remove**
  - **Download / Upload** — forces Site Sync transfer
- **Cherry-pick** toggles show only the selected subset.
- Full-text search across names / representations.

## Look Assigner (Maya)

Dedicated tool for pushing **Look** products onto assets. Pick asset → pick
look → apply. Backed by publish-side `look` products (shader + assignments
+ attrs extracted from a lookdev scene).

## Browser

`AYON → Browser` — web-backed browser for published products outside a
specific DCC. Good for producers / reviewers who don't need to open Maya.

## Tray Publisher

Standalone publisher that runs outside a DCC — useful for **plates**,
**references**, delivery assets, one-off imports. Same CCVEI pipeline, no
DCC scene.

Typical admins flow:

1. Configure Tray Publisher addon settings (which product types it can
   create, how variants resolve).
2. Artists drag files into Tray Publisher, pick product type + task, publish.

## Slater (slate generation)

Adds a branded title card to video reviews. Configured via the Slater
addon; integrates with Extract Review.

## Batch Delivery

Package + transcode approved versions for external clients. Profile-based.
Typically a producer or coordinator task.

## Site Sync (artist view)

- Loader / Scene Inventory show which site a representation is on.
- Click "Download" to pull from remote; artist keeps working, the
  background service transfers.
- If the studio runs a **studio-to-local** sync, artists rarely have to
  think about it.

## Admin / configuration quick map

| Thing you want to change | Where |
|--------------------------|-------|
| Where files go on disk | Studio Settings → Anatomy Presets → Roots; or per-project |
| Path templates | Project Anatomy → Templates |
| Which DCCs are available for a project | Project Settings → `applications` |
| Which product types a Creator offers | Core addon settings → Creator filtering profiles |
| Review output format | Core addon → Extract Review profiles |
| Per-user settings | Settings UI → user scope toggle |
| Developer mode for yourself | Admin marks you as developer; top-right checkbox |

## Troubleshooting patterns

- "My publish silently dropped something" → check Publisher's report panel;
  un-ticked optional plugins show there.
- "Loader shows the wrong version" → someone published since you opened the
  scene; close & reopen or use Scene Inventory → Update to Latest.
- "I can't see the tools" → DCC menu not installed; check host startup
  script / workfile opened without Ayon env (did you use `ayon.exe` or
  just launched Maya directly?).
- "Workfile-version mismatch between DCCs" → enable
  `ayon+settings://core/publish/CollectAnatomyInstanceData/follow_workfile_version`
  (FAQ).

## Sources

- <https://help.ayon.app/en/articles/4678978-getting-started-with-ayon-pipeline> — install + first login
- <https://help.ayon.app/en/articles/6207691-launcher> — launcher UI, tray icon, actions pane
- <https://help.ayon.app/en/articles/9624270-workfiles> — Workfiles tool
- <https://help.ayon.app/en/articles/1075843-creator-publisher> — Create vs Publish phases
- <https://help.ayon.app/en/articles/7070980-about-ayon-pipeline> — publishing principles + product types catalogue
- <https://help.ayon.app/en/articles/4345209-loader> — Loader UI
- <https://help.ayon.app/en/articles/9770233-scene-inventory> — Scene Inventory actions
- <https://help.ayon.app/en/articles/6062888-look-assigner> — Look Assigner
- <https://help.ayon.app/en/articles/4087894-browser> — Browser tool
- <https://help.ayon.app/en/articles/6447370-introduction-tray-publisher> — Tray Publisher intro
- <https://help.ayon.app/en/articles/7077398-using-slater-addon> — Slater
- <https://help.ayon.app/en/articles/1653353-introduction-ayon-batch-delivery> — Batch Delivery
- <https://help.ayon.app/en/articles/2590676-faq> — common FAQ (workfile version sync)
- <https://help.ayon.app/en/articles/6811988-working-with-maya-in-ayon> — representative DCC workflow article
- <https://help.ayon.app/articles/4902206-managing-projects> — project creation, deactivation, library projects
- <https://help.ayon.app/en/collections/1925761-dcc-integrations> — per-DCC article index
