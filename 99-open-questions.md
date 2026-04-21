# 99 — Open questions & known gaps

Things the KB is **uncertain about** or explicitly couldn't verify.
Re-check the live docs / repo source before trusting answers in these
areas.

## Broken / under-construction official docs

Returned 404 or "under construction" on the canonical docs site:

- `docs.ayon.dev/docs/system_introduction`
- `docs.ayon.dev/docs/dev_addon_distribution`
- `docs.ayon.dev/docs/dev_server_addon`
- `docs.ayon.dev/docs/artist_concepts`
- `docs.ayon.dev/docs/addon_core_artist`
- `docs.ayon.dev/docs/admin_settings_project_anatomy` → use
  `help.ayon.app/articles/3815114-project-anatomy`
- `docs.ayon.dev/docs/addon_sitesync_admin` → use
  `help.ayon.app/en/articles/3882937-site-sync`

`docs.ayon.dev/docs/dev_addon_intro` and
`docs.ayon.dev/docs/dev_addon_creation` render real content but still
carry the "under construction 🏗️" banner — treat their absolute
statements with caution.

## Areas only lightly covered

- **Frontend React components library** (`ayon-react-components`) —
  only catalogued via its Storybook URL. Read source for non-trivial
  frontend addons.
- **`OperationsSession` / entity hub** on `ayon-python-api` — mentioned
  but version-dependent.
- **Web Publisher** (`ayon+settings://web_publisher`) — artist side
  referenced; admin/dev details omitted.
- **Perforce workfile flow** — noted as an integration; not documented.
- **USD contribution workflow** — only linked; deep detail not in KB.
- **Per-DCC render-settings internals** (Maya's `lib_rendersettings.py`,
  Arnold/V-Ray/Redshift/RenderMan render-setup code) — only the image-
  prefix defaults are captured.

## Deliberately out of scope

- Full schema reference for every entity attribute — use `{server}/api`
  Swagger live.
- Full GraphQL schema — use `{server}/graphiql` live.
- UI theming (`docs.ayon.dev/docs/server_theme`).
- C++ API specifics (`ayon-cpp-api`) — link-only.

## Versions sampled during research

Inspected against `develop`:

- `ayon-maya` ~ 0.6.5
- `ayon-houdini` ~ 0.10.0
- `ayon-nuke` ~ 0.4.8
- `ayon-premiere` ~ 0.2.1+dev (manifest `ExtensionBundleVersion` 1.1.10)
- `ayon-deadline` ~ 0.7.1

Server `0.5.0+`, launcher `1.0.0-beta.6+`, `ayon-python-api` 1.2.16,
Ayon server Python 3.12, launcher/DCC Python 3.9.x.

These move; confirm against the bundle you're targeting.

## How to use this file

- Before citing a specific method / kwarg / class, grep current source
  and update the relevant KB file with a note if the shape has shifted.
- **When you confirm or debunk an item here, move it into the relevant
  numbered file** and delete the entry.
- Add new uncertainties here as they arise.

## Sources

- See `13-resources.md` for the canonical link index. DCC-specific
  sources are captured at the end of each numbered file (`20`–`24`).
