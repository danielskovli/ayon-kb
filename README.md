# Ayon Knowledge Base

A focused reference for working with **Ayon** (Ynput's open-source production
pipeline platform) — targeted at answering user ("how do I…") questions and
developer ("I want to build a plugin that…") questions alike.

Source material: `ynput.io/ayon`, `help.ayon.app`, `docs.ayon.dev`,
`github.com/ynput`. Initial draft 2026-04-21; cross-check with the live docs
before relying on any specific class name, import path, or endpoint — the
product moves fast. Every file ends with a **Sources** block pointing back to
the URLs used to build it.

## Index

### Concepts & architecture

| # | File | Contents |
|---|------|----------|
| 01 | [overview.md](01-overview.md) | What Ayon is, OpenPype lineage, terminology, repo map |
| 02 | [architecture.md](02-architecture.md) | Server stack, launcher, addons, ASH, deployment |
| 03 | [data-model.md](03-data-model.md) | Entities, relationships, settings scopes |

### Addon / plugin development

| # | File | Contents |
|---|------|----------|
| 04 | [addon-structure.md](04-addon-structure.md) | Repo layout, `package.py`, public/private, services/ |
| 05 | [server-addon.md](05-server-addon.md) | `BaseServerAddon`, settings, endpoints, web actions, frontend, settings UI widgets |
| 06 | [client-addon.md](06-client-addon.md) | `AYONAddon`, tray, plugin paths, CLI, launch hooks |
| 07 | [host-integration.md](07-host-integration.md) | DCC host impl — `HostBase`, interfaces, `install_host` |
| 08 | [publishing.md](08-publishing.md) | CCVEI pipeline, creators, validation errors, attr defs |
| 09 | [events-services.md](09-events-services.md) | Event topics, enroll pattern, ASH services |
| 15 | [loaders-inventory.md](15-loaders-inventory.md) | `LoaderPlugin`, utilities, container dict, scene inventory actions |

### Cross-cutting

| # | File | Contents |
|---|------|----------|
| 14 | [anatomy-templates.md](14-anatomy-templates.md) | Roots, templates, keys, presets, scoping |
| 16 | [site-sync-colorspace.md](16-site-sync-colorspace.md) | Sites + providers; OCIO + `ColormanagedPyblishPluginMixin` |

### APIs & ops

| # | File | Contents |
|---|------|----------|
| 10 | [apis.md](10-apis.md) | `ayon-python-api`, REST, GraphQL |
| 11 | [launcher-dev.md](11-launcher-dev.md) | Launcher CLI/env, bundles, dev mode, testing, contributing |
| 12 | [dcc-integrations.md](12-dcc-integrations.md) | DCC integration repos + "add a host" checklist |

### Playbooks

| # | File | Contents |
|---|------|----------|
| 17 | [user-workflows.md](17-user-workflows.md) | Artist flows: launcher → workfiles → create/publish → loader → inventory |
| 18 | [recipes.md](18-recipes.md) | 15 "I want to do X → do Y" developer recipes |

### Web admin

| # | File | Contents |
|---|------|----------|
| 19 | [admin-web.md](19-admin-web.md) | Projects, settings (color-coding, deep links), users + permissions, attributes, applications, bundles, events, services, Core levers |

### DCCs (deep dives)

| # | File | Contents |
|---|------|----------|
| 20 | [maya.md](20-maya.md) | Autodesk Maya — product catalog, object-set conventions, `cbId`, Look Assigner, Arnold/V-Ray/Redshift, render prefix |
| 21 | [houdini.md](21-houdini.md) | SideFX Houdini — `/out` ROPs, USD variants, loader HDAs, Deadline render targets |
| 22 | [nuke.md](22-nuke.md) | Foundry Nuke — Write convention, Templated Workfile Builder, farm targets, imageio |
| 23 | [premiere.md](23-premiere.md) | Adobe Premiere — CEP panel, `extension.zxp`, smart-layer rule, metadata bin |
| 24 | [deadline.md](24-deadline.md) | Thinkbox Deadline — two-job pattern, Houdini split, `GlobalJobPreLoad`, submitter classes |

### Meta

| # | File | Contents |
|---|------|----------|
| 13 | [resources.md](13-resources.md) | Canonical link index — user docs, dev docs, DCC articles, repos |
| 99 | [open-questions.md](99-open-questions.md) | Known doc gaps, unverified claims, out-of-scope items |

## Quick-start routes

### "How do I…" (artist / admin)
- **Launch a DCC for a task** → `17` §First-day workflow
- **Save a workfile** → `17` §Workfiles tool
- **Publish something** → `17` §Create / Publish
- **Pull a published asset into my scene** → `17` §Loader
- **Update loaded items** → `17` §Scene Inventory
- **Change where files go on disk** → `14` §Roots → `17` §Admin map
- **Understand product types** → `03` §Products + `17`
- **Create a new project / user / access group** → `19`
- **Upload an addon / create a bundle in the UI** → `19` §Bundles
- **Configure applications per project** → `19` §Applications
- **Interpret settings color-coding** → `19` §Settings

### "I want to build a plugin that…" (dev)
- **…is a full new addon from scratch** → `04` → `05` → `06` → `18` §1
- **…adds a new UI button that runs on the artist's machine** → `18` §2
- **…reacts to events** → `09` + `18` §3
- **…adds a new product type** → `08` + `18` §4 + `15`
- **…validates + self-repairs** → `18` §5
- **…downloads files outside a DCC** → `18` §6 + `10`
- **…adds a tab to the Ayon web UI** → `05` §Frontend scopes + `18` §7
- **…exposes a REST endpoint** → `05` §REST endpoints + `18` §8
- **…has host-scoped settings** → `05` §Settings UI + `18` §9
- **…handles colorspace correctly** → `16` + `18` §13
- **…runs on the farm** → `18` §14 + `24`

### DCC-specific
- **Maya publish patterns, product catalog** → `20`
- **Houdini USD contribution / loader HDAs** → `21`
- **Nuke Write / Templated Workfile Builder** → `22`
- **Premiere CEP install / smart layers** → `23`
- **Deadline two-job pattern / submitter classes** → `24`

### Deploy / run
- **Spin up a dev server** → `11` §Deploying a fresh server
- **Configure bundles** → `02` §Bundles + `11` §Bundles
- **Run a service** → `09` §Services + `02` §ASH

## Conventions

- Import paths and class names are taken verbatim from the docs where
  possible. Every file has a **Sources** block pointing at the exact URLs it
  drew from.
- Code blocks are **examples**, not copy-paste-ready templates. Versions
  move; confirm against the target bundle.
- "Legacy OpenPype" notes are included where modules still bear `openpype` /
  `pype` naming; Ayon is the rebranded successor, so expect mixed
  nomenclature in older code.
- `99-open-questions.md` tracks things I couldn't verify. If you confirm or
  debunk one of those, move the fact into the relevant numbered file.
