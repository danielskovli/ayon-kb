---
name: ayon-anatomy-templates
description: Ayon Project Anatomy for configuring file paths, resolving published paths, debugging "where did this file go", or changing project anatomy. Roots (per-platform paths for `{root[name]}`), template categories (work / publish / hero / staging / delivery with workfile template, publish template, hero template). Template keys (`{root[...]}`, `{project}`, `{folder[path]}`, `{task[short]}`, `{product[name]}`, `{version}`, `{@version}`, `{@frame}`, `{representation}`, `{ext}`). Optional segments `< {key} >`, uniqueness rules. Folder types, task types, statuses, tags, link types. Anatomy presets and scoping.
---

# Ayon Project Anatomy & templates

Anatomy is the per-project config that binds Ayon to your filesystem.
Modifying it mid-production risks broken references, lost files, and
status/tag mismatches — treat it as configuration-as-code.

## Roots

A root is a named storage location with **one path per OS**.

```
root "work":
  windows:  Z:/projects
  linux:    /mnt/studio/projects
  darwin:   /Volumes/projects
```

All platform paths must resolve to the **same logical folder**. Multiple
roots are allowed (e.g. `work` + `render`). Reference in templates as
`{root[name]}`.

## Template categories

| Category | Settings path | Purpose |
|----------|---------------|---------|
| **Work** | `ayon+settings://core/tools/Workfiles/workfile_template_profiles` | Where Workfiles tool saves |
| **Publish** | `ayon+settings://core/tools/publish/template_name_profiles` | Final published file location |
| **Hero** | `ayon+settings://core/tools/publish/hero_template_name_profiles` | "Latest/master" paths |
| **Staging** | `ayon+settings://core/tools/publish/custom_staging_dir_profiles` | Extract staging dir |
| **Delivery** | anatomy → Delivery | Desktop Browser deliveries |

Each profile resolves based on host + task type + product type.

## Template keys

### Context
- `{root[name]}`
- `{studio[name]}`, `{studio[code]}`
- `{project[name]}`, `{project[code]}`
- `{folder[name]}`, `{folder[type]}`, `{folder[path]}`
- `{hierarchy}` — folder path **excluding** folder name
- `{parent}` — parent folder name
- `{task[name]}`, `{task[type]}`, `{task[short]}` (e.g. `mdl`)
- `{app}` — application name (`maya`, `nuke`)
- `{user}`, `{user[name]}`, `{user[attrib][email]}`, `{user[data][...]}`

### Publish data
- `{product[name]}`, `{product[type]}`
- `{version}`
- `{representation}`
- `{ext}`, `{frame}`, `{output}`, `{comment}`
- `{originalDirname}`, `{originalBasename}`
- `{colorspace}`, `{udim}`

### Dynamic (DCC-specific)
- `{renderlayer}`, `{aov}` — Maya/Houdini render
- `{asset}` — legacy Houdini HDA/FBX

### Datetime
- `{d}`/`{dd}`, `{m}`/`{mm}`/`{mmm}`/`{mmmm}`, `{ddd}`/`{dddd}`,
  `{yy}`/`{yyyy}`, `{H}`/`{HH}`, `{h}`/`{hh}`/`{ht}`, `{M}`/`{MM}`,
  `{S}`/`{SS}`

## Syntax rules

- **Standard**: `{key}` — Python format syntax
- **Optional**: `< {key} >` — segment silently drops if key is absent.
  Nestable: `< {key1} <.{key2}> >`
- **Case-sensitive**: `{aov}`, `{Aov}`, `{AOV}` are distinct
- **Padded numerics**: always `@`-prefix — `{@frame}`, `{@version}` — to
  pick up studio padding config

## Uniqueness rules

### Work templates — **must resolve uniquely per task**

Minimum safe:

```
{root[work]}/{project[name]}/{folder[path]}/work/{task[name]}
```

Too-generic templates cause overwrites, version ambiguity, broken
automation. Add `{app}` when multiple DCCs share a task:

```
{root[work]}/{project[name]}/{folder[path]}/work/{task[name]}/{app}
```

### Publish templates — include `{version}` to avoid clashes

```
{root[work]}/{project[name]}/{folder[path]}/publish/{product[type]}/{product[name]}/{@version}
```

**Do not** put `{representation}` in the **directory** part — it breaks
the pipeline's resource logic. Put it in the **filename** and mark it
optional: `< /{representation} >` (covers farm renders that may omit it).

### Hero templates

Separate template so Hero files live alongside versioned ones without
conflict:

```
{root[work]}/{project[name]}/{folder[path]}/publish/{product[type]}/{product[name]}/hero/{product[name]}_hero<.{representation}>.{ext}
```

Integrators replace hero files atomically only after the versioned file
integrates successfully.

## Folder types, task types, statuses, tags, links

All defined **per project** in anatomy with names + colors.

- **Folder types**: `Asset`, `Shot`, `Sequence`, `Episode`, …
- **Task types**: `Modeling`, `Animation`, `Compositing`, … — carry short
  codes (`mdl`, `anm`, `cmp`) used in `{task[short]}`.
- **Statuses**: per-entity-type lists (folder, task, product, version,
  representation, workfile each separate). Display order = anatomy order.
- **Tags**: freeform labels; anatomy lists selectable ones.
- **Link types**:
  - `Generative` — input generates output (workfile → render)
  - `Reference` — input required (model → rig)
  - `Breakdown` — folder-to-folder (asset → shot)
  - `Template` — folder-to-folder fallback for workfile builders

## Attributes (deprecated)

The old Attributes tab is deprecated; use the Projects Dashboard instead.
Legacy projects may still use it as inherited default-attribute source.

## Anatomy presets

- Live in **Studio Settings** → Anatomy Presets.
- One is marked **primary**.
- New projects prompt for a preset at creation.
- **Not live-linked** — editing a preset doesn't update already-created
  projects.

## Scoping

- Studio Settings templates reference the **primary preset**.
- Project Settings templates reference that project's anatomy.
- A template name used in a profile must exist in the scope it's
  referenced from.

## Entity-naming rules

Folder / task auto-names derive from labels:

- Capitalisation: `all lower`, `ALL UPPER`, `Keep original`, `Title Case`,
  `title case Except The First`
- Separator: `_`, `-`, `.`, or none

## In code — resolve paths through Anatomy

Never string-substitute yourself. Use:

```python
from ayon_core.pipeline.anatomy import Anatomy
from ayon_core.pipeline.load import (
    get_representation_path, get_representation_path_by_names,
    get_representation_path_from_context,
)

path = get_representation_path(project, repre_entity, anatomy=Anatomy(project))
```

## Deeper reading

- `14-anatomy-templates.md` — full file with Sources
- Skill `ayon-loaders-inventory` — path resolution utilities

## Sources

- <https://help.ayon.app/articles/3815114-project-anatomy>
- <https://help.ayon.app/articles/3005275-core-addon-settings>
- <https://help.ayon.app/articles/4902206-managing-projects>

### Source repos

- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/pipeline/anatomy> — `Anatomy` class, template resolver
- <https://github.com/ynput/ayon-core/blob/develop/client/ayon_core/pipeline/load/utils.py> — `get_representation_path`, `get_representation_path_by_names`, `get_representation_path_from_context`
- <https://github.com/ynput/ayon-backend/tree/develop/ayon_server/entities> — project / folder / task entity definitions
- Anatomy preset definitions live in Studio Settings → Anatomy Presets (server-side; see `ayon-backend`)
