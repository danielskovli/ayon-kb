# 01 — Overview

## What Ayon is

Ayon (by **Ynput**) is an open-source **production pipeline platform** for animation
and VFX studios. Unlike task trackers (ftrack/Shotgrid/Kitsu), Ayon manages the
mechanics of the pipeline itself: file and version management, publish workflows,
DCC integration, quality enforcement, review/approval, multi-site file sync, and
background automation.

Key framing from ynput.io:
> "Manages files and versions, enforces standards, automates repetitive steps, and
> ties your DCC tools together."

Common user profile: studios with ~20+ team members, multi-project, often with
distributed/vendor workflows.

## Lineage

`pyblish` → `Avalon` → `OpenPype` → **Ayon**

- Publish logic is still **pyblish** under the hood — "every small addition is a
  pyblish plugin, even the code that registers products in Ayon Server."
- Many modules still use `openpype` / `pype` names (especially in older addons).
- `ayon-core` is the rebranded continuation of the main OpenPype codebase.

## Core product pieces

| Piece | What it is |
|-------|-----------|
| **Ayon Server** | Python backend (FastAPI + Postgres + Redis) + React frontend |
| **Ayon Launcher** | Desktop app on artist workstations. Bootstraps the pipeline, distributes addons & deps |
| **Addons** | Unit of extension. Ships server code + client code + optional frontend + services |
| **Bundles** | Server-side configuration: which addon versions + which launcher + which Python dep package |
| **ASH** (Ayon Service Host) | Small supervisor process that runs dockerised services against the server |
| **`ayon-python-api`** | Python client for REST + GraphQL |

## Terminology cheatsheet

- **Entity** — generic term for server objects: project, folder, task, product,
  version, representation, workfile, user, event.
- **Folder** — hierarchical node under a project (asset, shot, sequence, episode…).
  Replaces the legacy "asset" concept.
- **Task** — unit of work under a folder. Has a `taskType`.
- **Product** — what gets published from a task/folder. Has a `productType` (aka
  `family` in legacy code: `model`, `rig`, `render`, `workfile`, …).
- **Version** — a specific published version of a product. Monotonically versioned.
- **Representation** — a file-format output of a version (e.g. `.abc`, `.usd`,
  `.ma`). One version can have many representations.
- **Instance** — in-DCC object that will become a product at publish time.
- **Host** — DCC integration (`maya`, `nuke`, `blender`, `traypublisher`, …).
- **Bundle** — server-side named set: `{launcher_version, addon_versions,
  dependency_package, staging/production/dev}`.
- **Context** — `{project_name, folder_path, task_name}` that a launcher/DCC is
  currently operating in.

## Canonical repo map (`github.com/ynput`)

| Repo | Role |
|------|------|
| `ayon-backend` | Server API (Python, FastAPI, GraphQL, Postgres, Redis) |
| `ayon-frontend` | Web UI (React + TypeScript) |
| `ayon-docker` | Docker Compose for running the server |
| `ayon-launcher` | Desktop launcher (CX_Freeze, Poetry) |
| `ayon-core` | **Most important client addon** — base classes, pipeline, tools |
| `ayon-python-api` | Official REST/GraphQL Python client |
| `ayon-cpp-api` | C++ client |
| `ayon-usd` | USD addon (resolver, pipeline bits) |
| `ash` | Ayon Service Host — runs services |
| `ayon-dependencies-tool` | Builds Python dep packages for bundles |
| `ayon-react-components` | Shared React UI library (`@ynput/ayon-react-components`) |
| `ayon-maya`, `ayon-blender`, `ayon-houdini`, `ayon-nuke`, `ayon-3dsmax`, `ayon-hiero`, `ayon-tvpaint`, `ayon-unreal`, `ayon-fusion`, `ayon-aftereffects`, `ayon-photoshop`, `ayon-deadline`, … | Per-DCC and per-service addons |

Browse live list at <https://github.com/orgs/ynput/repositories>.

## Non-obvious facts

- Ayon is **multi-API**: REST + GraphQL (query-only) + Python + C++. Mutations go
  through REST; GraphQL is read-only.
- Settings have four scopes: **studio / project / project-site / studio-site**
  (see `03-data-model.md`).
- "Fire-and-forget" vs persistent events are both supported — persistent ones live
  in Postgres; f-a-f ones go over WebSocket (see `09-events-services.md`).
- Addons ship in one repo containing **both** server code and client code. They
  are built into a zip and uploaded to the server, which then distributes the
  client half through the launcher's bundle mechanism.

## Sources

- <https://ynput.io/ayon/> — product overview
- <https://help.ayon.app/en/articles/7070980-about-ayon-pipeline> — "About Ayon Pipeline" (publishing principles, product types, CCVEI overview)
- <https://docs.ayon.dev/docs/dev_addon_intro> — pyblish → Avalon → OpenPype → Ayon lineage; context + product/representation distinction
- <https://docs.ayon.dev/docs/dev_introduction> — developer entry point
- <https://github.com/ynput> — organisation landing
- <https://github.com/ynput/ayon-backend>, `/ayon-frontend`, `/ayon-core`, `/ayon-launcher`, `/ayon-python-api`, `/ayon-docker`, `/ash`, `/ayon-dependencies-tool`, `/ayon-react-components` — canonical repos
