# 19 — Ayon web administration

Everything an admin does from the Ayon web UI. Complements the dev-side
material in `05-server-addon.md`, `11-launcher-dev.md`, and
`14-anatomy-templates.md`.

## Top-level navigation

Two button clusters on the main bar:

**Left (production):**
- **Home** (`H+H`) — studio dashboard: "quick digest of tasks"
- **Projects** (`1`) — enter a specific project's folders/tasks/products

**Right (management):**
- User avatar dropdown — account, sessions, launcher downloads, sign out
- App grid (main menu)
- Inbox — notifications / mentions
- Help (`?`) — REST API, GraphQL Explorer, community links
- Chat bubble — AI assistant

**App grid:**

| Group | Entries | Shortcut |
|-------|---------|----------|
| Configuration | Studio Settings · Project Settings · Market | `S+S`, `P+P`, `M+M` |
| System | Event Viewer · Services · Server Actions | `E+E`, `V+V` |

Global search: `⌘K` / `Ctrl+K`.

## Projects

### Creating a project

Home page or Project Settings → `+` / `...`. At creation:

- **Project Name** — no spaces, **immutable** after creation
- **Project Code** — short identifier, editable
- **Anatomy Preset** — see `14-anatomy-templates.md`
- **Library Project Toggle** — flag as library (loadable cross-project)

### Deactivate, don't delete

> "It is highly recommended to deactivate real projects rather than
> deleting them, especially if they contain published work."

Deleted projects can only be recovered from backup. Deactivated projects
go grey under the **Show Archived** toggle and can be reactivated
later. `...` → Deactivate.

### Project folders (Pro/Studio)

Hierarchical organisation by client/type/status without affecting disk
location. Create with the Projects Actions menu or `F`. Features: labels,
icons, colours, move between folders, pinning (`/` to search), drag/drop.

### Project Overview page

Spreadsheet-like UI for editing project data directly:

- Double-click cells to edit; shift/ctrl/drag to multi-select
- Copy/paste across cells
- **Inherited attribute values in grey**
- Sort, pin, reorder, hide columns; resize rows
- Filters: attribute-based with suggested values; fuzzy search; folder
  sidebar filter; advanced filters with exclusions + "Match all"
- **Power Features**: group by task attributes (status, assignees, tags)
- Row selection opens detail panels
- Folder/task creation buttons
- Right-click context menus
- CSV export

## Settings system

Access: tray menu → Admin → Studio Settings (or `S+S`); project-scoped
values under Project Settings (`P+P`).

### Color coding

| Color | Meaning |
|-------|---------|
| **Grey** | AYON default (ships with current release) |
| **Green** | Studio default (your saved override; persists across updates) |
| **Orange** | Project-specific override |
| **Blue** | Unsaved changes (cleared on browser refresh) |

Orange propagates up the hierarchy so you can find changed settings by
scanning for colour.

### Override workflow

- **Studio default**: edit in System/Studio Settings tab, or choose
  "(Default)" in Project tab + save. Right-click: "add to studio
  default" / "remove from studio default".
- **Project override**: pick a project in Project Settings, modify,
  save.

### Deep-linking to a setting

`ayon+settings://<addon>/<path/through/schema>` resolves directly to a
UI field. Deep-links appear in docs and FAQ entries. Append
`?project=NAME` to jump to that project's override.

Examples:
- `ayon+settings://core/tools/Workfiles/workfile_template_profiles`
- `ayon+settings://deadline/deadline_server?project=myProject`
- `ayon+settings://premiere/auto_install_extension`

## Users & permissions

### Access levels (built-in roles)

| Role | Access |
|------|--------|
| **Admin** | Unrestricted; all projects, studio settings, bundles |
| **Manager** | All projects; limited studio settings; can't modify admins |
| **User** | Nothing by default — permissions entirely from assigned access groups |

### Access groups (built-in)

| Group | Typical permissions |
|-------|---------------------|
| **Artist** | Interact with pipeline; can't create/delete folders |
| **Freelancer** | Highly restricted; only sees assigned tasks |
| **Supervisor** | Broad project data access; no system settings |

Create additional groups as needed.

### Where permissions live (4 places)

1. **Studio Settings → Permissions** — global access-group rules
2. **Project Settings → Permissions** — per-project overrides
3. **Studio Settings → Users** — assign access level + default access group
4. **Project Settings → Project Access** — assign groups per user

**Critical**: "Default Project Access" applies only to **new** projects.
Existing projects require explicit assignment via Project Access. Users
not added to Project Access can't access the project regardless of
defaults.

### Partial-access options

- **Assigned** — only assigned tasks
- **Hierarchy** — folder and all contents
- **Children** — contents only (not the folder itself)

### Service accounts + API keys

Create a user with access level **Service** (details not in the user-
permissions article). Generate API keys from Studio Settings →
Users → API Keys. Keys authenticate with `X-Api-Key`; combine with
`X-as-user` to impersonate.

## Attributes

Studio-scope only (no per-project custom attributes). Studio Settings →
Attributes.

### Scope (7 entity types)

Project · Folder · Task · Product · Version · Representation · User

### Data types

| Type | Notes |
|------|-------|
| String | Optional length constraints, regex |
| Integer | Min/max |
| Decimal | Float |
| List of Strings | Multi-select |
| Boolean | Toggle |

### Enum attributes

String and list attributes can be **enumerated**. Each option has a
label, value, icon, and color for UI presentation.

### Inheritance

When enabled, child entities inherit parent values. Common pattern: set
at Project, inherited down through Folders to Tasks.

## Applications & tools

Studio Settings → Applications. Two-level structure:

### Application groups

Studio-wide settings applied to every variant:
- enable/disable toggle
- `host_name` (integration identifier: `maya`, `nuke`, …)
- group-level environment vars

### Application variants

Per-version:
- `name`, `label`
- per-platform executables (Windows / Linux / macOS paths)
- launch arguments (e.g. `--nukex`)
- version-specific env vars

**Linux isn't differentiated by distribution** — Ayon treats all Linux as
one platform.

### Non-integrated apps

Apps without an Ayon addon can still be registered. "They will only
launch but won't provide further AYON integration (menus, creators,
publishers)."

### Application filters

Profile-based rules in Project Settings pick which apps appear per
context. Match on task type; "all apps" or an explicit subset. **Always
keep a default profile** (blank task types) to prevent an empty
launcher.

### Env layering

Four levels, later wins:
1. Global Core settings
2. Application group
3. Application variant
4. Tool-specific

## Bundles & addons

Studio Settings → Bundles (and → Addons).

### Creating a bundle

`+ Add Bundle` or **Duplicate & Edit**. Configure:

- Bundle name + launcher version
- Addon list with specific versions
- Dev-bundle toggle (requires developer mode) + assigned user
- Compatibility checker runs against selected addons

After creation, most fields lock. Still editable: dependency packages,
some server-addon versions, pipeline mode assignment.

### Pipeline modes

| Mode | Purpose |
|------|---------|
| **Production** | Stable daily-work |
| **Staging** | Experimental — test without touching production |
| **Dev** | Developer-exclusive, live-code |

Only **one** bundle may be Production and only one Staging at a time.
Both flags can sit on the same bundle. Dev bundles can't flip to
Prod/Staging without first clearing Dev.

### Addons

Installed from:
- **AYON Market** — browse + install curated addons
- **Upload**: `.zip` files via buttons — Upload Addons, Upload
  Launcher, Upload Dependency Package.

Custom-addon uploads are restricted by default on Ayon Cloud; removal
of that restriction must be requested.

### Dependency packages

Python deps for addon client code. After bundle creation, assign one
package per OS. Produce either by:
- Updating pipeline (ships generic packages), or
- Building a custom package with `ayon-dependencies-tool`.

### Archiving

Right-click a bundle → Archive. **Show Archived** reveals them;
Unarchive from the context menu.

### Upgrade flow

Server Actions → Updating Pipeline → **Updating Pipeline to latest
Release** (or via Market). New addon versions then appear in the
Addons tab for inclusion in new bundles.

### Project bundles (Project Settings → Bundle)

Per-project override of which bundle applies. Useful when different
projects need different addon versions.

## Entity templates

Pre-defined folder/task hierarchies that can be stamped into a project.
Studio Settings → Entity Templates. Create a template, then apply it when
seeding a new project or spinning up a new asset/shot structure.

## Events (Event Viewer)

App grid → Event Viewer (`E+E`). Live + historical feed of server
events. Filters: topic, status, project, user, time range.

Key uses:
- Trace a publish (`entity.version.created`, `entity.representation.created`)
- Watch a service process jobs (status transitions)
- Debug an addon's custom topic
- Audit permission-sensitive changes (`entity.*.status_changed`)

## Services (`V+V`)

Monitor ASH-supervised services. Per service: configured replicas,
running status, env, logs. Add / remove / scale replicas. Each service
configuration is declared by an addon's `package.py` and toggled here.

## Market

App grid → Market. Browse Ayon + third-party addons, install with a
click, see compatibility info. Some entries may require a licence.

## Inbox

Personal notifications: assigned tasks, mentions in reviews/comments,
service events directed at you.

## Server deployment (admin-adjacent)

| Topic | Reference |
|-------|-----------|
| Local deployment | `help.ayon.app/en/articles/2293963-ayon-server-local-deployment` |
| Docker config options | `help.ayon.app/en/articles/4287499-ayon-docker-configuration-options` |
| Provisioning | `help.ayon.app/en/articles/4089565-ayon-server-provisioning` |
| Updating server | `help.ayon.app/en/articles/3808641-how-to-update-your-ayon-server` |
| ASH | `help.ayon.app/en/articles/7620452-ayon-service-host-ash` |
| Server secrets | `help.ayon.app/en/articles/3701069-ayon-server-secrets` |
| Licences | `help.ayon.app/en/articles/6592804-ayon-licenses` |
| Migrate OpenPype → Ayon | `help.ayon.app/en/articles/6657894-migrate-openpype-projects-to-ayon` |

For the Docker-based install: `github.com/ynput/ayon-docker` and
`11-launcher-dev.md`.

## Core addon settings — high-impact admin levers

The Core addon carries many studio-wide levers. Highlights:

| Lever | Location |
|-------|----------|
| Workfile templates | `core/tools/Workfiles/workfile_template_profiles` |
| Publish templates | `core/tools/publish/template_name_profiles` |
| Hero templates | `core/tools/publish/hero_template_name_profiles` |
| Staging dir profiles | `core/tools/publish/custom_staging_dir_profiles` |
| Creator filtering | `core/tools/Creator/...` — narrow creator list per host + task |
| Product name profiles | Product-name templates with `{family}`, `{task}`, `{variant}` + capitalisation |
| Extract Review | Multiple output formats via FFmpeg, filters by host + product |
| Extract Burnin | 6 default placeholders + custom template keys |
| Extract OIIO Transcode | Colorspace/display-space transcode via `oiiotool` |
| Integrate Product Group | Loader grouping by family/host/task type/task name |
| Open last workfile | Per task/host auto-load rules |
| Global OCIO config | Defaults to Ayon-shipped; per-project / per-host overrides |

## Keyboard shortcuts (common)

| Where | Keys |
|-------|------|
| Navigation | `H+H` home · `1` projects · `S+S` studio · `P+P` project · `M+M` market · `E+E` events · `V+V` services |
| Search | `⌘K` / `Ctrl+K` |
| Project folders | `F` create, `/` search |
| Loader | `Ctrl+G` group selected |

Full list: `help.ayon.app/en/articles/9085225-ayon-web-app-keyboard-shortcuts`.

## Sources

- <https://help.ayon.app/en/articles/5961925-navigating-ayon-server>
- <https://help.ayon.app/articles/4902206-managing-projects>
- <https://help.ayon.app/articles/7885519-project-overview-page>
- <https://help.ayon.app/en/articles/8317800-working-with-settings>
- <https://help.ayon.app/en/articles/3005275-core-addon-settings>
- <https://help.ayon.app/en/articles/6698141-user-management>
- <https://help.ayon.app/en/articles/2635883-user-permissions-and-project-access-groups>
- <https://help.ayon.app/en/articles/3945756-applications-and-tools>
- <https://help.ayon.app/en/articles/4636995-attributes>
- <https://help.ayon.app/en/articles/4644998-bundles-and-addons>
- <https://help.ayon.app/en/articles/0782429-project-bundles>
- <https://help.ayon.app/en/articles/2451163-entity-templates>
- <https://help.ayon.app/en/articles/7317917-ayon-market>
- <https://help.ayon.app/en/articles/2668013-ayon-services>
- <https://help.ayon.app/en/articles/6657894-migrate-openpype-projects-to-ayon>
- <https://help.ayon.app/en/articles/9085225-ayon-web-app-keyboard-shortcuts>
