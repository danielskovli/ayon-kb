# 13 — Resources

## Canonical docs

| Link | Covers |
|------|--------|
| <https://ynput.io/ayon/> | Product marketing, value prop, features |
| <https://help.ayon.app/> | User docs + admin docs (~162 articles) |
| <https://docs.ayon.dev/> | Developer docs |
| <https://docs.ayon.dev/ayon-core> | `ayon-core` generated docs |
| <https://docs.ayon.dev/ayon-python-api> | Python API reference |
| <https://docs.ayon.dev/ayon-cpp-api> | C++ API |
| <https://docs.ayon.dev/ayon-usd-resolver> | USD resolver |
| <https://components.ayon.dev/> | Storybook for the React component library |
| <https://playground.ayon.app/> | Public server (`/api` REST Swagger, `/graphiql` GraphQL explorer, `/explorer`) |

## Deep wikis (auto-generated)

| Link | Covers |
|------|--------|
| <https://deepwiki.com/ynput/ayon-backend> | Server internals |
| <https://deepwiki.com/ynput/ayon-frontend> | Frontend internals |

Useful for quick orientation when grepping. Treat as unofficial — verify against the live code.

## Help-center categories

From `help.ayon.app`:

| Category | Slug | Size |
|----------|------|------|
| General | `/collections/3934717-general` | 3 |
| Configuration | `/collections/0005002-configuration` | 7 |
| Production Tracking | `/collections/0376560-production-tracking` | 21 |
| Pipeline | `/collections/1069542-pipeline` | 36 |
| Server Integrations | `/collections/2471796-server-integrations` | 16 |
| DCC Integrations | `/collections/1925761-dcc-integrations` | 80 |
| Server | `/collections/6984978-server` | 19 |

## GitHub

- Organisation: <https://github.com/ynput>
- Core repos: see `01-overview.md#canonical-repo-map`
- DCC integrations: see `12-dcc-integrations.md`

## Community

- Ynput Community forum (official discussion + feature proposals):
  <https://community.ynput.io/> — hosted on the Ynput company domain,
  **not** `community.ayon.app`. See AGENTS.md § Domain map.
- Discord — invite via community forum.
- Roadmap + changelog + feedback portal — linked from `help.ayon.app/`.

## Useful URLs when running locally

Default `ayon-docker` install:

- Frontend: `http://localhost:5000/`
- REST Swagger: `http://localhost:5000/api`
- GraphiQL: `http://localhost:5000/graphiql`
- OpenAPI JSON: `http://localhost:5000/openapi.json`
- Addons listing: `http://localhost:5000/api/addons`
- Bundles: `http://localhost:5000/api/bundles`

## Cheat-sheet — "where do I look for X?"

| Question | Start with |
|----------|-----------|
| "Does the server support Y?" | `{server}/api` (Swagger) |
| "Can I query Z?" | `{server}/graphiql` |
| "How does addon A work?" | `github.com/ynput/ayon-<a>/client/ayon_<a>/` |
| "What fields does entity E have?" | `schemas/` in `ayon-backend` repo |
| "What pyblish plugins run for family F?" | grep `families = ["F"]` in `ayon-core` + relevant host addon |
| "How do I add a menu item?" | `ITrayAddon` (launcher) or host `startup/` (DCC) |
| "How do I react to an event?" | `09-events-services.md` — enroll pattern |
| "How is auth done?" | `10-apis.md` — bearer token or `X-Api-Key` |

## Versions seen during research

- Ayon server: referenced as 0.5.0+ (dev-mode minimum)
- Ayon launcher: 1.0.0-beta.6+ (dev-mode minimum)
- `ayon-python-api`: 1.2.16
- Ayon server Python: 3.12
- Ayon launcher/DCC Python: 3.9.x (VFX Platform CY2022)

These are reference points from the docs — always re-check against the
current target server/bundle.

## Extended link index

### help.ayon.app — user / admin articles

| Topic | URL |
|-------|-----|
| Project Anatomy (templates, roots, types) | <https://help.ayon.app/articles/3815114-project-anatomy> |
| Managing Projects | <https://help.ayon.app/articles/4902206-managing-projects> |
| Core Addon Settings | <https://help.ayon.app/articles/3005275-core-addon-settings> |
| About AYON Pipeline | <https://help.ayon.app/en/articles/7070980-about-ayon-pipeline> |
| Getting Started with AYON Pipeline | <https://help.ayon.app/en/articles/4678978-getting-started-with-ayon-pipeline> |
| Launcher | <https://help.ayon.app/en/articles/6207691-launcher> |
| Launcher – Admin Notes | <https://help.ayon.app/en/articles/2308428-launcher-admin-notes> |
| Workfiles | <https://help.ayon.app/en/articles/9624270-workfiles> |
| Creator / Publisher | <https://help.ayon.app/en/articles/1075843-creator-publisher> |
| Loader | <https://help.ayon.app/en/articles/4345209-loader> |
| Scene Inventory | <https://help.ayon.app/en/articles/9770233-scene-inventory> |
| Look Assigner | <https://help.ayon.app/en/articles/6062888-look-assigner> |
| Browser | <https://help.ayon.app/en/articles/4087894-browser> |
| Site Sync | <https://help.ayon.app/en/articles/3882937-site-sync> |
| Tray Publisher – user | <https://help.ayon.app/en/articles/5385780-working-with-ayon-tray-publisher> |
| Tray Publisher – TDs | <https://help.ayon.app/en/articles/0866874-tray-publisher-for-tds> |
| Slater / Slate Editor | <https://help.ayon.app/en/articles/7077398-using-slater-addon> · <https://help.ayon.app/en/articles/2723500-slate-editor> |
| Batch Delivery | <https://help.ayon.app/en/articles/1653353-introduction-ayon-batch-delivery> |
| Advanced Burnins | <https://help.ayon.app/en/articles/7585314-advanced-burnins> |
| Web Publisher | <https://help.ayon.app/en/articles/4834329-web-publisher> |
| Batch Ingest | <https://help.ayon.app/en/articles/8591610-batch-ingest> |
| OpenUSD Overview | <https://help.ayon.app/en/articles/6167506-why-and-what-is-openusd> |
| OpenUSD Resolver | <https://help.ayon.app/en/articles/4862376-ayon-openusd-resolver> |
| FAQ | <https://help.ayon.app/en/articles/2590676-faq> |

### help.ayon.app — DCC-integration articles (artist flows)

Every supported DCC has a "Working with X in AYON" article and usually a
settings article. See full index in <https://help.ayon.app/en/collections/1925761-dcc-integrations>. Highlights:

| DCC | "Working with" | Settings |
|-----|----------------|----------|
| Maya | <https://help.ayon.app/en/articles/6811988-working-with-maya-in-ayon> | <https://help.ayon.app/en/collections/8127361-maya-addon-settings> |
| Blender | <https://help.ayon.app/en/articles/2501380-working-with-blender-in-ayon> | <https://help.ayon.app/en/articles/6836825-configure-blender-addon> |
| Houdini | <https://help.ayon.app/en/articles/2777507-working-with-houdini-in-ayon> | <https://help.ayon.app/en/articles/0982414-configure-houdini-addon> |
| Nuke | <https://help.ayon.app/en/articles/4670584-working-with-nuke-in-ayon> | <https://help.ayon.app/en/articles/2833967-nuke-addon-settings> |
| 3ds Max | <https://help.ayon.app/en/articles/6074628-working-with-3ds-max-in-ayon> | <https://help.ayon.app/en/articles/3959010-3ds-max-addon-settings> |
| Unreal | <https://help.ayon.app/en/articles/2228078-working-with-unreal-in-ayon> | <https://help.ayon.app/en/articles/8279245-unreal-addon-settings> |
| AfterEffects | <https://help.ayon.app/en/articles/9429657-working-with-after-effects-in-ayon> | <https://help.ayon.app/en/articles/4515672-aftereffects-addon-settings> |
| Photoshop | <https://help.ayon.app/en/articles/7712931-working-with-photoshop-in-ayon> | <https://help.ayon.app/en/articles/0146725-photoshop-addon-settings> |
| Deadline | <https://help.ayon.app/en/articles/8138091-working-with-deadline-in-ayon> | <https://help.ayon.app/en/articles/5372986-configure-deadline-addon> |

### docs.ayon.dev — developer docs index

| Topic | URL |
|-------|-----|
| Developer intro | <https://docs.ayon.dev/docs/dev_introduction> |
| Requirements | <https://docs.ayon.dev/docs/dev_requirements> |
| Launcher | <https://docs.ayon.dev/docs/dev_launcher> |
| Dev mode | <https://docs.ayon.dev/docs/dev_dev_mode> |
| Addon intro | <https://docs.ayon.dev/docs/dev_addon_intro> |
| Addon creation | <https://docs.ayon.dev/docs/dev_addon_creation> |
| Host implementation | <https://docs.ayon.dev/docs/dev_host_implementation> |
| Publishing | <https://docs.ayon.dev/docs/dev_publishing> |
| Event system | <https://docs.ayon.dev/docs/dev_event_system> |
| Colorspace | <https://docs.ayon.dev/docs/dev_colorspace> |
| Testing | <https://docs.ayon.dev/docs/dev_testing> |
| Contributing | <https://docs.ayon.dev/docs/dev_contribute> |
| Python API | <https://docs.ayon.dev/docs/dev_api_python> · <https://docs.ayon.dev/ayon-python-api/> |
| REST API | <https://docs.ayon.dev/docs/dev_api_rest> |
| GraphQL API | <https://docs.ayon.dev/docs/dev_api_graphql> |
| C++ API | <https://docs.ayon.dev/docs/dev_api_cpp> · <https://docs.ayon.dev/ayon-cpp-api/> |
| USD resolver | <https://docs.ayon.dev/docs/dev_api_usd_resolver> · <https://docs.ayon.dev/ayon-usd-resolver/> |
| Server introduction | <https://docs.ayon.dev/docs/server_introduction> |
| Server API architecture | <https://docs.ayon.dev/docs/server_api_architecture> |
| Server theme | <https://docs.ayon.dev/docs/server_theme> |
| Deadline (dev) | <https://docs.ayon.dev/docs/dev_deadline> |
| React components | <https://docs.ayon.dev/docs/dev_components_react> · <https://components.ayon.dev/> |

### Known-broken / under-construction docs pages

These returned 404 or "under construction" during research (2026-04). Check
before citing them:

- `docs.ayon.dev/docs/system_introduction`
- `docs.ayon.dev/docs/dev_addon_distribution`
- `docs.ayon.dev/docs/dev_server_addon`
- `docs.ayon.dev/docs/artist_concepts`
- `docs.ayon.dev/docs/addon_core_artist`
- `docs.ayon.dev/docs/admin_settings_project_anatomy` (use help.ayon.app instead)
- `docs.ayon.dev/docs/addon_sitesync_admin`

### Sources

This file is the master link index; the URLs above are themselves the
sources. They were collected via WebFetch across `ynput.io/ayon`,
`help.ayon.app`, `docs.ayon.dev`, and `github.com/ynput` on 2026-04-21.
