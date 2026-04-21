# 03 — Data model

## Entity hierarchy

```
Project
└── Folder              (hierarchical; folderType = asset, shot, sequence, …)
    ├── Folder …        (arbitrary nesting)
    └── Task            (taskType = modeling, animation, comp, …)
        └── Product     (productType = model, rig, render, workfile, …)
            └── Version (integer, monotonically increasing per product)
                └── Representation  (one per file format: abc, usd, mov, …)
```

### Projects
- Top-level container. Key fields: `name`, `code`, `library`, `attrib`, `data`,
  per-project lists of `folderTypes`, `taskTypes`, `statuses`, `tags`,
  `productTypes`.
- Identified by `name` (the canonical project id).

### Folders
- Replaces the legacy "asset" concept from OpenPype.
- Path-based: `folder.path` is `/seq010/sh010` style.
- Has a `folderType` (studio-configurable list). Common: `Asset`, `Shot`,
  `Sequence`, `Episode`.
- Can contain sub-folders.
- Carries `attrib` (attributes like `fps`, `resolutionWidth`, `clipIn/Out`,
  `handleStart/End`) and `data` (free-form JSON).

### Tasks
- Live under a folder. Not nestable.
- Have `taskType` (studio-configurable).
- Have `assignees` (users), `status`.
- The publish context `{project, folder, task}` normally resolves to a single
  task.

### Products
- What can be published from a task/folder.
- `productType` replaces legacy `family`. Common: `model`, `rig`, `look`,
  `animation`, `render`, `workfile`, `plate`, `pointcache`, `camera`, `usd`.
- Named with a studio-defined template (`productName` — e.g.
  `modelMain`, `renderFarmV2Beauty`).

### Versions
- Integer version numbers start at 1 and increment per product.
- Have `author`, `status`, `attrib` (frameStart/frameEnd, etc.), `data`, `tags`.

### Representations
- Files attached to a version. Each representation has:
  - `name` (usually the extension/short name: `abc`, `usd`, `exr`)
  - one or more `files[]` with `path`, `size`, `hash`, `size` info
  - `attrib` / `data` for metadata like template info, site availability

### Users, Access Groups
- Server has `users`, `accessGroups`, `roles`. Service accounts (API key
  authenticated) use `X-Api-Key` and can impersonate with `X-as-user`.

### Events
- First-class entity. See `09-events-services.md`.

## In-DCC counterpart: instances & containers

- **Instance** (a.k.a. `CreatedInstance`) — created by a **Creator** plugin.
  Represents "a thing in this scene that, when I publish, will become a Product
  + Version + Representations." Lives in scene metadata (varies by DCC).
- **Container** — a *loaded* product+version+representation. Tracked in scene
  metadata by the host integration. Retrieved via
  `host.get_containers()`.

Typical `CreatedInstance` dict keys:
- `id` — constant `"pyblish.avalon.instance"`
- `instance_id` — UUID
- `family` / `productType`
- `creator_identifier`
- `variant` (e.g. `Main`, `High`)
- `productName` — resolved via template with variant/task/folder
- `asset` / `folderPath`, `task`
- `active`
- `creator_attributes` — plugin-specific bag

## Settings scopes

Settings are strongly typed (pydantic models). There are four scopes:

| Scope | Stored where | Applies to |
|-------|---------------|-----------|
| **studio** | server, global | whole studio |
| **project** | server, per project | single project (overrides studio) |
| **studio-site** | local workstation | this machine, globally |
| **project-site** | local workstation | this machine + project |

On a `SettingsField` you set `scope=["studio", "project"]` etc. If omitted, the
setting applies across all relevant scopes.

## Attribs

`attrib` is a typed property bag tied to entity type. Studio attributes can be
extended via the server's attribute management. Common built-in attribs:

- Folder: `fps`, `resolutionWidth`, `resolutionHeight`, `pixelAspect`,
  `clipIn`, `clipOut`, `handleStart`, `handleEnd`, `frameStart`, `frameEnd`
- Task: `assignee`, `priority`
- Version: `fps`, `source`, `intent`, `comment`, `families`
- Representation: `template`, `extension`, `site`

## IDs

All server entities have UUIDs. Most REST endpoints accept either a UUID or a
human name/path (project by `name`, folder by id or path, task by id).

## Status, Tags, Types

Per project, configurable lists of:

- Statuses (`Not Ready`, `In Progress`, `Pending Review`, `Approved`, …)
- Tags (free-form colored labels)
- Folder types
- Task types
- Product types

Status changes emit events like `entity.folder.status_changed` with
`payload/newValue` which services can key off (see `09-events-services.md`).

## Sources

- <https://docs.ayon.dev/docs/dev_addon_intro> — context (project/folder/task) and product (representations, product types) distinction
- <https://docs.ayon.dev/docs/dev_addon_creation> — four settings scopes (studio / project / studio-site / project-site)
- <https://docs.ayon.dev/docs/dev_publishing> — `CreatedInstance` structure and key fields
- <https://help.ayon.app/articles/3815114-project-anatomy> — folder types, task types, statuses, tags, link types, attributes
- <https://help.ayon.app/articles/4902206-managing-projects> — project creation, deactivation, library projects
- <https://help.ayon.app/en/articles/7070980-about-ayon-pipeline> — product types overview
- `{server}/api` — Swagger schema (authoritative for entity fields)
- `{server}/graphiql` — GraphQL schema explorer
