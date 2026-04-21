# 14 — Anatomy & templates

**Anatomy** is the per-project configuration that binds Ayon to your
filesystem. It defines:

- **Roots** — where files live on disk (per platform)
- **Templates** — how paths are constructed from entities
- **Folder & task types**, **statuses**, **tags**, **link types**, **attributes**

Modifying anatomy **mid-production** risks breaking references, lost files, and
status/tag mismatches. Treat it as configuration-as-code.

## Roots

A root is a named storage location with **one path per OS**. Example:

```
root "work":
  windows:  Z:/projects
  linux:    /mnt/studio/projects
  darwin:   /Volumes/projects
```

Multiple roots are allowed — e.g. a fast local `work` root and a shared
`render` root. Every root must resolve to the same logical folder on every
platform (the pipeline trusts this equivalence).

Roots are referenced in templates as `{root[name]}`, e.g. `{root[work]}`.

## Templates

Templates are Python format strings resolved against an entity context.

### Template categories

| Category | Settings path | What it builds |
|----------|---------------|----------------|
| **Work** | `ayon+settings://core/tools/Workfiles/workfile_template_profiles` | Where the Workfiles tool saves scene files |
| **Publish** | `ayon+settings://core/tools/publish/template_name_profiles` | Final published file destinations |
| **Hero** | `ayon+settings://core/tools/publish/hero_template_name_profiles` | "Latest" / master version paths |
| **Staging** | `ayon+settings://core/tools/publish/custom_staging_dir_profiles` | Temp dir for extracted files during publish |
| **Delivery** | (anatomy → Delivery) | Used by the Desktop Browser's delivery actions |
| **Others** | per-addon | Addons can register additional templates |

Each profile resolves based on host + task type + product type.

### Available keys

**Context:**
- `{root[name]}`
- `{studio[name]}`, `{studio[code]}`
- `{project[name]}`, `{project[code]}`
- `{folder[name]}`, `{folder[type]}`, `{folder[path]}`
- `{hierarchy}` — folder path excluding folder name
- `{parent}` — parent folder name
- `{task[name]}`, `{task[type]}`, `{task[short]}` (e.g. `mdl`)
- `{app}` — application name (e.g. `maya`)
- `{user}`, `{user[name]}`, `{user[attrib][email]}`, `{user[data][...]}`

**Publish data:**
- `{product[name]}`, `{product[type]}`
- `{version}` — product version
- `{representation}` — representation name
- `{ext}` — file extension
- `{frame}` — frame number (sequence files)
- `{output}` — Extract Review output profile
- `{comment}` — subversion tag (workfiles) or publish comment
- `{originalDirname}`, `{originalBasename}` — original file info
- `{colorspace}`, `{udim}`

**Dynamic publish (DCC-specific):**
- `{renderlayer}`, `{aov}` — Maya/Houdini render
- `{asset}` — legacy Houdini HDA/FBX

**Datetime:**
- `{d}`/`{dd}`, `{m}`/`{mm}`/`{mmm}`/`{mmmm}`, `{ddd}`/`{dddd}`,
  `{yy}`/`{yyyy}`, `{H}`/`{HH}`, `{h}`/`{hh}`/`{ht}`, `{M}`/`{MM}`,
  `{S}`/`{SS}`

### Syntax

- **Standard:** `{key}` — Python format syntax.
- **Optional:** `< {key} >` — segment silently drops if the key is absent.
  Nestable: `< {key1} <.{key2}> >`.
- **Case-sensitive:** `{aov}`, `{Aov}`, `{AOV}` are distinct keys.
- **Zero-padded numeric:** always use `@`-prefix for these —
  `{@frame}`, `{@version}` — to pick up the studio's padding config.

### Uniqueness requirements

**Work templates** must resolve uniquely per task. Minimum safe form:

```
{root[work]}/{project[name]}/{folder[path]}/work/{task[name]}
```

Too-generic templates cause overwrites, version ambiguity, and broken
automation. Add `{app}` when multiple DCCs share a task:

```
{root[work]}/{project[name]}/{folder[path]}/work/{task[name]}/{app}
```

**Publish templates** include `{version}` so iterations don't clash. Prefer:

```
{root[work]}/{project[name]}/{folder[path]}/publish/{product[type]}/{product[name]}/{@version}
```

**Do not** put `{representation}` in the **directory** part — it breaks the
pipeline's resource logic. Use it in the **filename** and mark it optional:
`< /{representation} >` (covers farm render outputs that may omit it).

### Hero templates

Hero is "latest" / "master". Define a separate template so Hero files can
live alongside versioned ones without conflict:

```
{root[work]}/{project[name]}/{folder[path]}/publish/{product[type]}/{product[name]}/hero/{product[name]}_hero<.{representation}>.{ext}
```

Hero writes are guarded — integrators replace the hero file atomically only
once the versioned file has been successfully published.

## Folder & task types

- Defined **per project** in anatomy.
- Each has a name + color.
- Folder types: usually `Asset`, `Shot`, `Sequence`, `Episode`, …
- Task types: `Modeling`, `Animation`, `Compositing`, …
- Task types get **short codes** (`mdl`, `anm`, `cmp`) used in `{task[short]}`.

## Statuses, tags, link types

- **Status** lists are per-entity-type (folder, task, product, version,
  representation, workfile — each can have its own list).
- Display order in anatomy = display order in UI.
- **Tags** are freeform labels; anatomy lists which tags are selectable.
- **Link types** define typed relationships:
  - `Generative` — input generates output (workfile → render, model → cache)
  - `Reference` — input required (model → rig, footage → comp)
  - `Breakdown` — folder-to-folder (asset → shot)
  - `Template` — folder-to-folder fallback for workfile builders

## Attributes (deprecated)

The old "Attributes" section under anatomy is deprecated in favour of the
Projects Dashboard. Legacy projects may still use it as a source of default
attribute values inherited by folders.

## Anatomy presets

- Presets live in **Studio Settings** → Anatomy Presets.
- One preset is marked **primary**.
- New projects prompt for a preset at creation.
- **Presets are not live-linked** — changing a preset doesn't update
  already-created projects. Copy-and-apply is manual.

## Scoping

- **Studio Settings** templates reference the **primary** anatomy preset.
- **Project Settings** templates reference that project's anatomy.
- A template name used in a profile must exist in the scope it's referenced
  from, or the profile's selector won't find it.

## Entity-naming rules

Anatomy defines how folder/task **names** are auto-derived from their labels:

- Capitalization: `all lower`, `ALL UPPER`, `Keep original`, `Title Case`,
  `title case Except The First`
- Separator: `_`, `-`, `.`, or none

## When building a publish plugin

- Use `ayon_core.pipeline.anatomy.Anatomy` to resolve paths.
- Use `get_representation_path(project_name, repre_entity, anatomy=...)` and
  friends (see `15-loaders-inventory.md`) rather than doing string
  substitution yourself.
- Never hardcode separators; respect `{root[...]}`.

## Sources

- <https://help.ayon.app/articles/3815114-project-anatomy> — master reference (roots, templates, keys, uniqueness, presets, scoping)
- <https://help.ayon.app/articles/3005275-core-addon-settings> — workfile/publish profile examples
- <https://help.ayon.app/articles/4902206-managing-projects> — preset selection at project creation
- <https://help.ayon.app/en/articles/2590676-faq> — example of `ayon+settings://` deep-links
- `ayon_core.pipeline.anatomy` — in-code resolver (check `ayon-core` repo)
