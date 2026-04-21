# 02 — Architecture

## Big picture

```
              ┌─────────────────────────┐
              │   Ayon Server (docker)  │
              │                         │
 artists  ──► │  FastAPI + GraphQL + WS │ ◄── services (ASH)
              │  Postgres  +  Redis     │
              │  React frontend         │
              └─────┬───────┬───────────┘
                    │       │
           bundles/ │       │ REST / GraphQL / WS
           installers       │
                    │       │
             ┌──────▼───────▼────┐
             │   Ayon Launcher   │   (workstation)
             │   + addons (client │
             │     halves)        │
             └──────┬─────────────┘
                    │ subprocess + env
             ┌──────▼──────┐
             │ DCC (Maya,  │
             │ Nuke, etc.) │
             └─────────────┘
```

## Server (ayon-backend)

- **Language**: Python 3.12.
- **Stack**: FastAPI, Strawberry (GraphQL), asyncpg (Postgres), aioredis (Redis),
  gunicorn/uvicorn (optionally granian).
- **Main dirs** in repo:
  - `ayon_server/` — core application
  - `api/` — REST endpoints
  - `schemas/` — data models (pydantic)
  - `addons/` — addon loading/serving
  - `cli/` — `aycli`, `dbshell`
  - `maintenance/` — ops utilities
- **Frontend** (`ayon-frontend`) is a separate repo: React + TS + Vite + Redux
  Toolkit + RTK Query, styled-components, ARC (Ayon React Components).
- Default dev listen: `http://localhost:5000`.
- **Code gen**: frontend generates TS types + RTK Query hooks from OpenAPI and
  `.graphql` files (`gen/openapi-config.ts`). Generated files are then enhanced
  with cache tags & transforms.

## Launcher (ayon-launcher)

- Desktop app. Built with **CX_Freeze**, env managed with **Poetry**.
- Platforms: Windows (`ayon.exe`, `ayon_console.exe`), Linux, macOS.
- Responsibilities:
  1. Authenticate against server; fetch active bundle.
  2. Download addons + Python dependency package for that bundle into local
     storage dir.
  3. Bootstrap `ayon-core` and call into its CLI / UI.
  4. Manage tray UI, launch DCCs with the right env.
- Storage dirs (defaults):
  - Windows: `%LOCALAPPDATA%\Ynput\AYON`
  - Linux: `~/.local/share/AYON`
  - macOS: `~/Library/Application Support/AYON`
- Registering the shim: run `ayon init-ayon-launcher` once after install so it
  becomes the OS-wide handler.

See `11-launcher-dev.md` for full CLI flags and env vars.

## Addon system

A single "addon" is a repo that may contain up to four sides:

1. **Server side** (`server/`) — loaded by the server, adds settings, REST
   endpoints, event handlers, web actions, frontend mount points.
2. **Client side** (`client/<name>/`) — Python package distributed to
   workstations via launcher, loaded inside DCCs.
3. **Frontend** (`frontend/dist/`) — static web app embedded in the server UI
   (iframe) under configured scopes.
4. **Services** (`services/`) — dockerised long-running processes run by ASH.

Not every addon has all four. A DCC integration is usually server (just settings)
+ client (the bulk). A production tracker sync addon is usually server +
services. See `04-addon-structure.md` and `05` / `06`.

## Bundles

A **bundle** is a named, server-defined snapshot of:

- launcher version to use
- every addon + the version of it that should be active
- Python **dependency package** (a pre-built bundle of wheels for
  `ayon-core` + DCC-side Python libs)
- a flag — `production`, `staging`, or `dev`

The launcher asks the server for the bundle that applies to it (prod by default;
staging with `--use-staging`; dev bundle with `--use-dev`). It then downloads
whatever it doesn't already have locally. A bundle is what makes an Ayon install
reproducible across all artists.

Dev bundles bypass version pinning — a developer's bundle can be pointed at a
**local path** for a client addon so edits show up on launcher restart.

## ASH — Ayon Service Host

- Small Python process (Python 3.10+) in repo `ynput/ash`.
- Reads the server's services table and keeps the declared services running
  (usually via Docker).
- Services get `AYON_SERVER_URL` + `AYON_API_KEY` from ASH.
- Typical services: `ayon-ftrack`, `ayon-shotgrid`, `ayon-kitsu`, review pollers.

## Deployment

- Canonical production deployment is **docker-compose** from `ayon-docker`.
- Demo projects: `make demo` / `manage.ps1 demo` (commercial, big-episodic,
  big-feature).
- Port 5000 in dev by default; behind nginx in prod.
- Admin docs for installation: <https://help.ayon.app/>.

## Data flow example (publish)

1. Artist saves a workfile in Maya.
2. `ayon-core` + `ayon-maya` run a **pyblish** session.
3. Creators find/mark instances in the scene.
4. Collectors/validators/extractors run (CCVEI).
5. Integrator uploads files + calls REST `POST /api/projects/.../versions` and
   `.../representations` via `ayon-python-api`.
6. Server persists → emits `entity.version.created` and
   `entity.representation.created` events.
7. Services + other addons react via the **enroll** pattern.

## Sources

- <https://docs.ayon.dev/docs/server_introduction> — Python 3.12, FastAPI, Strawberry (GraphQL), asyncpg, aioredis, gunicorn/uvicorn
- <https://docs.ayon.dev/docs/server_api_architecture> — frontend codegen (REST + GraphQL via RTK Query)
- <https://docs.ayon.dev/docs/dev_launcher> — launcher responsibilities, bundles, storage dirs
- <https://github.com/ynput/ayon-backend> — server repo (ayon_server/, api/, schemas/, addons/, cli/)
- <https://github.com/ynput/ayon-docker> — docker-compose deployment, demo projects
- <https://github.com/ynput/ash> — ASH supervisor (Python 3.10, docker-sock mount, stable hostname)
- <https://deepwiki.com/ynput/ayon-backend> — unofficial deep wiki
