---
name: ayon-pipeline-basics
description: Ayon fundamentals — use for orientation questions "what is Ayon", "how is Ayon architected", "what's a folder / product / version / bundle", "is this the same as OpenPype", "where does the launcher fit", "what are bundles", and for disambiguating OpenPype-era vs Ayon-era terminology (family vs product, avalon, pype). Product overview, server / launcher / addon architecture. Entity model: Project → Folder → Task → Product → Version → Representation. Bundles (prod/staging/dev). pyblish / Avalon / OpenPype / Ayon lineage. ASH (Ayon Service Host) and `ayon-python-api` basics.
---

# Ayon pipeline basics

Ayon is Ynput's open-source **production pipeline platform** for animation
and VFX studios. Lineage: pyblish → Avalon → OpenPype → **Ayon**. Publishing
is still pyblish under the hood; `ayon-core` is the renamed OpenPype
codebase. Expect some modules and classes to still carry `openpype` / `pype`
/ `avalon` names — that's not legacy debt, it's the current product.

## Core pieces

| Piece | What it is |
|-------|------------|
| **Ayon Server** | Python 3.12 / FastAPI / Strawberry GraphQL / Postgres / Redis + React frontend (Vite + Redux Toolkit + ARC) |
| **Ayon Launcher** | Desktop app (CX_Freeze + Poetry). Bootstraps pipeline, downloads addons + deps |
| **Addons** | Extension unit. One repo = server half + client half + optional frontend + optional services |
| **Bundles** | Server-side snapshot: `{launcher_version, addon_versions, dep_package, prod/staging/dev}` |
| **ASH** (Ayon Service Host) | Python 3.10 supervisor running dockerised services against the server |
| **`ayon-python-api`** | REST + GraphQL Python client |

## Data model

```
Project
└── Folder              (hierarchical; folderType = Asset/Shot/Sequence/Episode…)
    ├── Folder …        (arbitrary nesting)
    └── Task            (taskType = Modeling/Animation/Comp/…)
        └── Product     (productType = model/rig/render/workfile/…)
            └── Version (integer, increments per product)
                └── Representation   (one per format — abc, usd, exr, mov, …)
```

**Settings scopes** (4): `studio`, `project`, `studio-site`, `project-site`.

**Events** are first-class: `entity.<type>.created` / `.updated` /
`.deleted` / `.status_changed`; plus `log.*`, `server.restart_requested`,
`settings.changed`. Persistent (stored) or fire-and-forget (WebSocket only).

## Vocabulary traps

| Looks like | Means |
|---|---|
| "folder" | Ayon entity (asset/shot/sequence/episode), **not** a filesystem folder |
| "product" | Publishable thing. Legacy code uses `family` for the same concept |
| "family" | Old word for `productType` — still pervasive in `ayon-core` |
| "asset" | Legacy word for `folder`. Still in help docs |
| `OpenPypePyblishPluginMixin` | Current class — not a bug, do not "fix" |
| `openpype:container-2.0` | Current scene-container schema string |
| `avalon` | Deep-legacy package name, pre-OpenPype |

## Canonical repo map (`github.com/ynput`)

| Repo | Role |
|------|------|
| `ayon-backend` | Server API |
| `ayon-frontend` | Web UI |
| `ayon-docker` | Docker-compose deployment |
| `ayon-launcher` | Desktop launcher |
| `ayon-core` | **Main client addon** — base classes, pipeline, tools |
| `ayon-python-api` | REST/GraphQL client |
| `ayon-cpp-api`, `ayon-usd` | C++ client, USD resolver + pipeline |
| `ash` | Service Host supervisor |
| `ayon-dependencies-tool` | Builds Python dep packages for bundles |
| `ayon-react-components` | Shared React UI (`@ynput/ayon-react-components`) |
| `ayon-addon-template` | Boilerplate for new addons |
| `ayon-<host>` | Per-DCC: maya, blender, houdini, nuke, 3dsmax, unreal, aftereffects, photoshop, hiero, fusion, tvpaint, resolve, cinema4d, flame, … |
| `ayon-ftrack`, `ayon-shotgrid`, `ayon-kitsu`, `ayon-deadline`, `ayon-slack`, `ayon-jira`, `ayon-perforce` | Production tracker / farm / notification integrations |

## Non-obvious facts

- Ayon is **multi-API**: REST + GraphQL (query-only) + Python + C++ + WebSocket.
  Mutations go through REST; GraphQL is read-only.
- Addons ship **server and client code in the same repo**, zipped and
  uploaded to the server; the server then distributes the client half to
  each launcher as part of a bundle.
- A single addon can supply up to four sides: `server/`, `client/`,
  `frontend/`, `services/` — not all are required.
- Dev mode lets a developer point the launcher at a **local path** for a
  client addon — edit-reload without re-upload. Server-side code changes
  still require re-upload.

## Data flow example (publish)

1. Artist saves a workfile in a DCC.
2. `ayon-core` + `ayon-<host>` run a pyblish session.
3. Creators find/mark instances in the scene.
4. Collectors → Validators → Extractors → Integrators (CCVEI).
5. Integrator uploads files + registers products/versions/reps via REST.
6. Server emits `entity.version.created` / `.representation.created`.
7. Services react via the **enroll** pattern.

## Deeper reading

- `01-overview.md`, `02-architecture.md`, `03-data-model.md` in the KB root
  — full human-readable references with Sources blocks.
- More specific topics have dedicated skills: `ayon-addon-development`,
  `ayon-publishing`, `ayon-anatomy-templates`, `ayon-events-services`, etc.

## Sources

- <https://ynput.io/ayon/>
- <https://help.ayon.app/en/articles/7070980-about-ayon-pipeline>
- <https://docs.ayon.dev/docs/dev_addon_intro>
- <https://docs.ayon.dev/docs/server_introduction>
- <https://docs.ayon.dev/docs/dev_launcher>
- <https://github.com/ynput> — organisation landing

### Source repos

- <https://github.com/ynput/ayon-backend> — server (Python, FastAPI, GraphQL, Postgres, Redis)
- <https://github.com/ynput/ayon-frontend> — web UI (React + Vite + RTK Query)
- <https://github.com/ynput/ayon-core> — main client addon (pipeline, tools, publish glue)
- <https://github.com/ynput/ayon-launcher> — desktop launcher
- <https://github.com/ynput/ayon-python-api> — Python client
- <https://github.com/ynput/ayon-docker> — compose deployment
- <https://github.com/ynput/ash> — service host
- <https://github.com/ynput/ayon-dependencies-tool> — dep package builder
- <https://github.com/ynput/ayon-react-components> — ARC component lib
- <https://github.com/ynput/ayon-addon-template> — addon boilerplate
- <https://github.com/orgs/ynput/repositories?q=ayon-> — live list of all addons
