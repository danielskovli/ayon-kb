---
name: ayon-admin-web
description: Ayon web-UI administration — creating/managing projects (anatomy preset, library toggle, deactivate-don't-delete), Studio vs Project Settings (with color coding grey/green/orange/blue and `ayon+settings://` deep-links), users + access levels (admin/manager/user) + access groups (Artist/Freelancer/Supervisor), attributes (7 entity types, enums, inheritance), applications + variants + filters + env layering, bundles (prod/staging/dev, upload addons, dependency packages, archiving), event viewer, services page, project folders, project overview spreadsheet, keyboard shortcuts, server deployment admin topics. Use when answering admin/ops questions about the Ayon web UI, guiding a new-project setup, configuring permissions, uploading addons, or tuning Core addon settings.
when_to_use: Triggered by "create project", "manage project", "deactivate project", "library project", "project overview", "studio settings", "project settings", "access group", "permissions", "service account", "API key", "attribute", "enum attribute", "applications addon", "application variant", "upload addon", "bundle", "production bundle", "staging bundle", "dev bundle", "dependency package", "event viewer", "services page", "ayon+settings://", "color coding", "migrate OpenPype".
---

# Ayon web administration

Everything an admin does in the Ayon web UI. For anatomy specifically
see `ayon-anatomy-templates`; for addon/bundle *development* see
`ayon-launcher-dev` and `ayon-addon-development`.

## Top-level nav

**Left (production):** Home (`H+H`), Projects (`1`).
**Right (management):** user avatar · app grid · Inbox · Help · AI chat.

**App grid:**

| Group | Entries | Shortcut |
|-------|---------|----------|
| Configuration | Studio Settings · Project Settings · Market | `S+S`, `P+P`, `M+M` |
| System | Event Viewer · Services · Server Actions | `E+E`, `V+V` |

Global search: `⌘K` / `Ctrl+K`.

## Projects

### Creating one
- **Project Name** — no spaces, **immutable**
- **Project Code** — short identifier, editable
- **Anatomy Preset** — selects templates + folder/task/status types
- **Library Project Toggle** — loadable cross-project

### Deactivate, don't delete
Admin doctrine: **deactivate** (reversible) rather than delete (data
loss). `...` → Deactivate. "Show Archived" reveals deactivated ones.

### Project folders (Pro/Studio plans)
Hierarchical grouping of projects without affecting disk. `F` create,
`/` search. Pin, move, recolour.

### Project Overview
Spreadsheet UI for folders/tasks:
- Double-click to edit; shift/ctrl/drag multi-select; copy/paste
- Inherited attrs shown in grey
- Sort · pin · hide · reorder · resize rows
- Filters: attribute (with suggestions), fuzzy task search, folder
  sidebar, advanced filters with exclusions + "Match all"
- **Power Features** group by status / assignees / tags
- CSV export

## Settings — color coding

| Color | Meaning |
|-------|---------|
| **Grey** | AYON default |
| **Green** | Studio default (saved override) |
| **Orange** | Project-specific override (propagates upward as breadcrumbs) |
| **Blue** | Unsaved changes |

### Deep-linking

`ayon+settings://<addon>/<path/through/schema>?project=NAME`

Examples:
- `ayon+settings://core/tools/Workfiles/workfile_template_profiles`
- `ayon+settings://deadline/deadline_server?project=showA`
- `ayon+settings://premiere/auto_install_extension`
- `ayon+settings://core/publish/CollectAnatomyInstanceData/follow_workfile_version`

## Users & permissions

### Access levels (built-in roles)

| Role | Reach |
|------|-------|
| **Admin** | Unrestricted |
| **Manager** | All projects; limited studio settings; can't modify admins |
| **User** | Nothing by default; permissions from Access Groups |

### Access groups (built-in)

| Group | Typical |
|-------|---------|
| **Artist** | Pipeline use; no folder create/delete |
| **Freelancer** | Only explicitly-assigned tasks |
| **Supervisor** | Broad project data, no system settings |

### Permission config lives in **four places**
1. Studio Settings → Permissions — global group rules
2. Project Settings → Permissions — per-project overrides
3. Studio Settings → Users — default access level + group
4. **Project Settings → Project Access** — per-user, per-project groups

Critical gotcha: "Default Project Access" applies **only to new
projects**. Existing projects need explicit Project Access entries —
users without an entry can't see that project.

### Partial access options
`Assigned` · `Hierarchy` · `Children`

### Service accounts & API keys
Service-level users get API keys. Use `X-Api-Key` (+ optional
`X-as-user` for impersonation).

## Attributes

Studio-scope only. Add at Studio Settings → Attributes.

- **Scope** (pick one or more of 7): Project · Folder · Task · Product
  · Version · Representation · User
- **Types**: String · Integer · Decimal · List of Strings · Boolean
- **Enum** (on String / List): labelled values with icons + colors
- **Inheritance**: children inherit parent value when enabled
- **Validation**: regex, min/max length, numeric min/max

## Applications & tools

Studio Settings → Applications. Two levels:

- **Application group** — studio-wide env + `host_name` + enable
- **Application variant** — per-version executables (Win/Linux/macOS),
  launch args (e.g. `--nukex`), per-variant env

Non-integrated apps can still be registered (no menus/creators).

**Application Filters**: Project Settings profile — which apps appear
per task type. **Always** keep a default (blank task types) profile, or
the launcher list may be empty.

Env layering (later wins): Core → Group → Variant → Tool.

**Linux isn't differentiated** by distribution — one platform.

## Bundles

Studio Settings → Bundles.

### Creating
`+ Add Bundle` or **Duplicate & Edit**. Pick name, launcher version,
per-addon versions, dev toggle. After creation, most fields lock;
dep packages + some server addon versions + pipeline mode remain
editable.

### Modes
- **Production** — one at a time
- **Staging** — one at a time
- **Dev** — developer-exclusive, live-code; can't flip to
  Prod/Staging without clearing the Dev flag first

Both Prod + Staging flags can sit on the same bundle.

### Addons

Install via:
- **Market** — curated addons
- **Upload** — `.zip` files via Upload Addons / Upload Launcher /
  Upload Dependency Package

Custom uploads are restricted by default on Ayon Cloud subscriptions;
request removal from support.

### Dependency packages

Python deps for addon client code. Per-OS assignment. Either ships with
pipeline updates or built via `ayon-dependencies-tool`.

### Archive / unarchive

Right-click. "Show Archived" toggle reveals them.

### Project bundles

Project Settings → Bundle. Per-project override of which bundle applies.

## Entity templates

Studio Settings → Entity Templates. Pre-defined folder/task hierarchies
stampable into projects or asset families.

## Events (Event Viewer, `E+E`)

Live + historical event feed. Filter by topic / status / project /
user / time. Key for:
- Tracing a publish (`entity.version.created`)
- Watching services process jobs
- Debugging addon topics
- Auditing status changes

## Services (`V+V`)

ASH-supervised services — per-service replicas, status, logs, env.
Declared by addon `package.py`.

## Market

App grid → Market. Browse Ayon + third-party addons; install with one
click; some require a licence.

## Core addon — high-impact admin levers

| Lever | Settings path |
|-------|---------------|
| Workfile templates | `core/tools/Workfiles/workfile_template_profiles` |
| Publish / Hero templates | `core/tools/publish/template_name_profiles` · `hero_template_name_profiles` |
| Staging dir profiles | `core/tools/publish/custom_staging_dir_profiles` |
| Creator filtering | `core/tools/Creator/...` |
| Product name profiles | Templates with `{family}`, `{task}`, `{variant}` + capitalisation |
| Extract Review | FFmpeg output formats + host/product filters |
| Extract Burnin | Placeholders + custom template keys |
| Extract OIIO Transcode | Colorspace/display conversion |
| Integrate Product Group | Loader grouping by family/host/task |
| Open last workfile | Per task/host rules |
| Global OCIO config | Defaults to Ayon's; per-project / per-host overrides |

## Keyboard shortcuts

`H+H` home · `1` projects · `S+S` studio · `P+P` project · `M+M`
market · `E+E` events · `V+V` services · `⌘K` search · `F` create
folder · `/` folder search · `Ctrl+G` group (loader).

## Server deployment

For the admin corners of ops: local deploy, docker config,
provisioning, updating server, ASH, secrets, licences, OpenPype →
Ayon migration. Start from the KB root file.

## Deeper reading

- `19-admin-web.md` — full file with Sources
- Skill `ayon-anatomy-templates` — anatomy deep-dive
- Skill `ayon-launcher-dev` — dev-side bundle/dep-package flow

## Sources

- <https://help.ayon.app/en/articles/5961925-navigating-ayon-server>
- <https://help.ayon.app/articles/4902206-managing-projects>
- <https://help.ayon.app/articles/7885519-project-overview-page>
- <https://help.ayon.app/en/articles/8317800-working-with-settings>
- <https://help.ayon.app/en/articles/3005275-core-addon-settings>
- <https://help.ayon.app/en/articles/2635883-user-permissions-and-project-access-groups>
- <https://help.ayon.app/en/articles/3945756-applications-and-tools>
- <https://help.ayon.app/en/articles/4636995-attributes>
- <https://help.ayon.app/en/articles/4644998-bundles-and-addons>

### Source repos

- <https://github.com/ynput/ayon-backend> — server source (settings/users/addons/attributes/bundles surfaces)
- <https://github.com/ynput/ayon-backend/tree/develop/ayon_server/entities> — entity definitions incl. Project, User, AccessGroup
- <https://github.com/ynput/ayon-backend/tree/develop/ayon_server/settings> — settings model framework, `SettingsField`
- <https://github.com/ynput/ayon-backend/tree/develop/ayon_server/addons> — addon loader / manager
- <https://github.com/ynput/ayon-frontend> — Web UI (project overview, settings, bundles pages)
- <https://github.com/ynput/ayon-docker> — Docker deployment
- <https://github.com/ynput/ayon-applications> — Applications addon (defines groups + variants)
- <https://github.com/ynput/ash> — service supervisor (for the Services page)
