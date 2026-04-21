---
name: ayon-user-workflows
description: Artist- and admin-facing Ayon workflows (user perspective, not developer). Use when answering "how do I publish / load / open a workfile", "how does the Ayon UI work", "how do I launch Maya through Ayon", "where are the addon settings", or UI-driven artist/admin questions. Covers launcher UI / navigation, Workfiles tool, Creator + Publisher UI (Create vs Publish phases), Loader, Scene Inventory UI, Tray Publisher, Slater, Batch Delivery, Site Sync from the user view, managing projects (creating a project, library project), admin configuration quick-map.
---

# Ayon user workflows (artist / admin)

For developer plumbing, see the other `ayon-*` skills. This skill covers
what artists and admins actually do in the UI.

## First-day workflow (launching a task)

1. **Install** — download the launcher from the studio's Ayon server, run
   installer; first run asks for server URL + credentials.
2. **Tray icon** appears. Color indicates mode (production / staging).
3. Click tray → **open Launcher window**.
4. **Select**: Project → Folder → Task. "Show only my tasks" filter
   available.
5. **Actions pane** shows DCCs/tools configured for this task. Click to
   launch.
6. The DCC boots with the right env + (by default) your last workfile.
   Right-click the app icon to start empty or pick a specific workfile.

Per-context buttons:

- **Explore Here** — file explorer at work dir
- **Show in AYON** — jumps to the context in the web UI
- **Terminal** — shell with env pre-loaded

## Workfiles tool

`AYON → Workfiles` replaces File → Open / Save As.

- **Open**: select from list (newest first) → **Open** or double-click.
- **Save As**: optional sub-version tag (e.g. `_mattepainting`) → confirm.
  `Ctrl+S` does increment-save.
- **Auto version bump** v001 → v002 → …, manual override supported.
- Artist notes attachable per workfile.
- Publishing may trigger further auto-bump depending on settings.

Versions are **per task**, governed by the anatomy's work template.

## Create / Publish

`AYON → Publisher` — two phases of the same tool.

### Create

1. Select scene content.
2. Open Creator (AYON menu or Publisher's *Create* tab).
3. Pick a product type (Model / Camera / Render / …).
4. Enter **variant** (`Main`, `High`, `V01`).
5. Click **Create**.

Scene side-effect varies per DCC:
- **Maya**: `objectSet` with metadata on custom attrs
- **Nuke**: Write node with metadata in hidden knobs
- **Resolve**: timeline markers with metadata in notes

Every asset is expected to have a `Main` product; variants are optional.

### Publish

Two-panel UI:
- **Left**: instances about to publish
- **Right**: create + publish attributes

Run **dry-run** to execute Collect + Validate only. Fix validation errors
→ re-run → click **play** to finish Extract + Integrate.

Error types:
- **`PublishXmlValidationError`** — plugin-specific help text / repair action
- **`PublishValidationError`** — the plugin's own message
- **`KnownPublishError`** — unrecoverable, retry later

## Loader

`AYON → Load…` — bring published products into the current scene.

- Filters: product name (text) + product type (checklist)
- Version column: double-click → version picker. Hero shows source
  version in brackets, e.g. `[v003]`.
- **Enable Grouping**: `Ctrl+G` — groups persist per project.
- Site widget (with Site Sync): per-representation availability + download.
- Library projects are selectable — cross-project loads supported.

The action performed by "Load" depends on which loader plugin the artist
picks (multiple may be compatible).

## Scene Inventory

`AYON → Manage → Inventory` — status + lifecycle for every loaded
container.

- Outdated items go red.
- Actions: **Update to Latest** · **Change to Hero** · **Set Version** ·
  **Switch Asset** · **Remove** · **Download / Upload** (Site Sync).
- **Cherry-pick** toggles to show only selected subset.
- Full-text search across names / representations.

## Look Assigner (Maya)

Pushes **Look** products onto assets. Pick asset → pick look → apply.
Backed by publish-side `look` products.

## Browser

`AYON → Browser` — web-backed browser for published products outside a
DCC. Good for producers / reviewers.

## Tray Publisher

Standalone publisher that runs outside a DCC — plates, references,
delivery assets, one-off imports. Same CCVEI pipeline, no DCC scene.

Admin flow:
1. Configure Tray Publisher addon settings (allowed product types,
   variant resolution).
2. Artists drag files in, pick product type + task, publish.

## Slater — slate generation

Brand card on video reviews. Configured via the Slater addon; integrates
with Extract Review.

## Batch Delivery

Package + transcode approved versions for external clients. Profile-based.
Usually a producer/coordinator task.

## Site Sync (artist view)

- Loader / Scene Inventory show which site holds each representation.
- Click **Download** to pull from remote; background service transfers.
- If studio runs a **studio → local** sync, artists rarely think about it.

## Admin configuration quick-map

| Goal | Location |
|------|----------|
| Change where files go on disk | Studio Settings → Anatomy Presets → Roots; or per-project |
| Edit file path templates | Project Anatomy → Templates |
| Make DCCs available for a project | Project Settings → `applications` |
| Restrict which product types a Creator offers | Core addon → Creator filtering profiles |
| Configure review output format | Core addon → Extract Review profiles |
| Per-user setting overrides | Settings UI → user scope toggle |
| Give yourself developer mode | Admin marks user as developer; top-right toggle |

## Managing projects

- Create via Home → `+` / `...`. Name (immutable, no spaces), code (short,
  editable), anatomy preset, library-project toggle.
- **Deactivate** rather than delete — published work may be referenced.
- "Show Archived" reveals deactivated projects.
- Pro/Studio subscription unlocks **project folders** (hierarchical
  organisation without affecting disk).

## Troubleshooting patterns

- **Publish silently dropped something** → check Publisher's report
  panel; un-ticked optional plugins show there.
- **Loader shows wrong version** → someone published since you opened;
  reopen or use Scene Inventory → Update to Latest.
- **Ayon menu missing in DCC** → DCC launched outside Ayon env. Use
  `ayon.exe` / launcher, not the DCC icon directly.
- **Workfile-version mismatch across DCCs** → enable
  `ayon+settings://core/publish/CollectAnatomyInstanceData/follow_workfile_version`.

## Deeper reading

- `17-user-workflows.md` — full file with Sources
- `13-resources.md` — per-DCC "Working with X" help articles

## Sources

- <https://help.ayon.app/en/articles/4678978-getting-started-with-ayon-pipeline>
- <https://help.ayon.app/en/articles/6207691-launcher>
- <https://help.ayon.app/en/articles/9624270-workfiles>
- <https://help.ayon.app/en/articles/1075843-creator-publisher>
- <https://help.ayon.app/en/articles/7070980-about-ayon-pipeline>
- <https://help.ayon.app/en/articles/4345209-loader>
- <https://help.ayon.app/en/articles/9770233-scene-inventory>
- <https://help.ayon.app/en/articles/6447370-introduction-tray-publisher>
- <https://help.ayon.app/en/articles/1653353-introduction-ayon-batch-delivery>
- <https://help.ayon.app/en/articles/2590676-faq>
- <https://help.ayon.app/articles/4902206-managing-projects>
