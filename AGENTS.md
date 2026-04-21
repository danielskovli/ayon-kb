# AGENTS.md

Brief for any agent (or new human collaborator) working in this repo.
Read this before touching files or answering Ayon questions.

## What this repo is

A curated knowledge base about **Ayon** — Ynput's open-source VFX /
animation pipeline platform, successor to OpenPype. Two surfaces:

- **Flat markdown at repo root** (`01-*.md` → `18-*.md`, `99-*.md`) —
  human-readable reference with Sources blocks.
- **Skills at `.claude/skills/<name>/SKILL.md`** — agent-facing
  distillations that auto-load when relevant. Full detail for each
  topic lives here; the KB files are the upstream source.

## Vocabulary traps (never defer these)

| Looks like | Means |
|---|---|
| "folder" | Ayon entity (asset/shot/sequence/episode), **not** a filesystem folder |
| "product" | Publishable thing. Legacy code uses `family` for the same concept |
| "family" | Old word for `productType` — pervasive in `ayon-core` |
| "asset" | Legacy word for `folder`. Still in help docs |
| `OpenPypePyblishPluginMixin` | Current class — not a bug, don't "fix" |
| `openpype:container-2.0` | Current scene-container schema string |
| `avalon` | Deep-legacy package name, pre-OpenPype |

## Domain map (don't guess URLs)

Ayon-the-product and Ynput-the-company own different domains. Pattern-matching from one to the other produces dead links.

| Domain | Owner | Scope |
|---|---|---|
| `ayon.app` / `ayon.dev` / `help.ayon.app` / `docs.ayon.dev` | Product | Docs, help center, settings deep-links (`ayon+settings://`) |
| `ynput.io` | Company | Marketing, blog |
| `community.ynput.io` | Company | Official community forum (**not** `community.ayon.app`) |
| `github.com/ynput` | Company | All source repos — `ayon-core`, `ayon-backend`, per-DCC addons |

When unsure, check `13-resources.md` before fetching. Don't synthesise a URL from a pattern.

## Skills catalog

Let the descriptions route you. Each `ayon-*` skill auto-loads on
relevance; full bodies live at `.claude/skills/<name>/SKILL.md`.

### Reference (auto + manual) — core

| Skill | Scope |
|-------|-------|
| `ayon-pipeline-basics` | Overview, architecture, data model, bundles, OpenPype lineage |
| `ayon-addon-development` | Addon repo layout, `package.py`, server side, client side |
| `ayon-host-integration` | `HostBase` + interfaces, `install_host`, DCC bootstrap |
| `ayon-publishing` | pyblish CCVEI, creators, validation errors, attribute defs |
| `ayon-loaders-inventory` | `LoaderPlugin`, load utilities, container dict, inventory actions |
| `ayon-events-services` | Event topics, enroll pattern, ASH services |
| `ayon-anatomy-templates` | Roots, templates, keys, presets |
| `ayon-site-sync-colorspace` | Site Sync + OCIO / `ColormanagedPyblishPluginMixin` |
| `ayon-apis` | `ayon-python-api`, REST, GraphQL |
| `ayon-launcher-dev` | Launcher CLI, bundles, dev mode, testing, contributing |
| `ayon-dcc-integrations` | Per-DCC repo map + artist-help articles |
| `ayon-user-workflows` | Artist/admin "how do I…" |
| `ayon-recipes` | 15 "I want to build X → do Y" patterns |

### Reference — web admin + DCC deep-dives

| Skill | Scope |
|-------|-------|
| `ayon-admin-web` | Web-UI admin: projects, users + permissions, attributes, applications, bundles, event viewer, services, Core levers |
| `ayon-maya` | Autodesk Maya — product catalog, object-set conventions, `cbId`, Look Assigner, Arnold / V-Ray / Redshift |
| `ayon-houdini` | SideFX Houdini — `/out` ROPs, USD variants, loader HDAs, Deadline render targets |
| `ayon-nuke` | Foundry Nuke — Write convention, Templated Workfile Builder, farm targets, imageio |
| `ayon-premiere` | Adobe Premiere — CEP panel, `extension.zxp`, smart-layer rule, metadata bin |
| `ayon-deadline` | Thinkbox Deadline — two-job pattern, Houdini split, `GlobalJobPreLoad`, submitter classes |

### Task (manual only — `disable-model-invocation: true`)

| Skill | Purpose |
|-------|---------|
| `/ayon-verify <symbol>` | Verify a claimed class/method/kwarg against live `ayon-*` source before recommending code |
| `/ayon-new-addon <name>` | Scaffold a new addon from `ynput/ayon-addon-template` |

## Verification policy

**No guessing, no assuming.** Every Ayon-specific claim — class names,
method signatures, kwargs, setting keys, env vars, endpoint paths,
file locations, workflow steps, version behaviour — must be grounded
in the KB (a skill or numbered file) or verified against live source
**before** you state it. Do not synthesise answers from training
data, from OpenPype / Avalon / pyblish prior art, or from generic
pipeline patterns — Ayon has drifted from all of them, and a
plausible-sounding guess is worse than "I don't know yet" because it
looks authoritative.

Docs and code drift too. Workflow:

1. **Answer from the relevant skill** (auto-loaded) or its KB file.
2. **If the claim isn't in the KB** — or is in the KB but about a
   specific class / method / signature / kwarg / setting key / env
   var / endpoint path — verify against live source before
   recommending:
   - `github.com/ynput/ayon-core` — client classes, pipeline, plugins
   - `github.com/ynput/ayon-backend` — server classes, schemas, endpoints
   - `github.com/ynput/ayon-python-api` — REST/GraphQL client
   - per-DCC addon repo — host-specific behaviour

   Use `/ayon-verify <symbol>` to formalise this.
3. **If live source contradicts the skill/KB**, trust the source and
   update the KB file in place (plus its Sources block).
4. **If you can't verify, say so plainly** — don't fill the gap. Add
   an entry to `99-open-questions.md` and flag the uncertainty in
   your answer, or ask the user.

## KB conventions

- Flat numbered markdown at repo root; skills live under `.claude/skills/`.
- `README.md` is the human index — update when adding/renaming root files.
- Every root KB file ends with `## Sources` citing canonical URLs.
- Skills end with `## Sources` too, pointing to the same URLs they
  derive from.
- `99-open-questions.md` tracks unverified claims. Confirmed items
  graduate into the numbered file they belong to.
- **Extend existing content before creating new content**. New root file
  = new topic. New skill = new domain.
- Known-broken doc URLs are catalogued in `99-open-questions.md` — don't
  cite them.

## Working style

- This repo is a reference notebook. When the user asks an Ayon question,
  extend the KB in place (root file + corresponding skill if applicable).
- When adding to a root file, keep its structure (sections match the
  skill's structure so updates can flow both ways).
- Do not invent URLs. If a URL isn't in a Sources block or in
  `13-resources.md`, verify it first.
- Don't duplicate material between the skill and its KB file beyond what
  the skill needs to be standalone — if the topic is deepening, add to
  the KB file first, then refresh the skill.

## Live fallbacks

- REST schema: `{server}/api` · GraphQL: `{server}/graphiql`
- OpenAPI JSON: `{server}/openapi.json`
- User docs: <https://help.ayon.app/>
- Dev docs: <https://docs.ayon.dev/>
- Organisation: <https://github.com/ynput>
