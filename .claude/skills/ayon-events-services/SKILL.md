---
name: ayon-events-services
description: Ayon event system and services — for reacting to server events, writing a background worker, subscribing to topics from a server addon, or deploying a dockerised Ayon service. Persistent events vs fire-and-forget (`store=False`), WebSocket event stream, topic taxonomy (`entity.folder.created`, `entity.version.created`, `entity.folder.status_changed`, `log.info`, `log.*`, `settings.changed`). `dispatch_event`, `depends_on` chains. The enroll pattern for idempotent services (`POST /api/enroll`). ASH (Ayon Service Host) as supervisor. Declaring services from an addon's `package.py`.
---

# Ayon events & services

## Two kinds of events

| Flavour | Stored? | Delivery | Use |
|---------|---------|----------|-----|
| **Persistent** | Postgres | WebSocket + poll | jobs, audit, depends-on chains |
| **Fire-and-forget** | No | WebSocket only | UI hints, "reload this view" |

Pass `store=False` to dispatch a fire-and-forget event.

## Event fields

| Field | Notes |
|-------|-------|
| `topic` | Required (e.g. `entity.folder.created`) |
| `id` | Auto UUID |
| `hash` | Optional — dedup key per sender |
| `summary` | Small JSON broadcast to clients ("headline") |
| `payload` | Full JSON, REST-only (not over WS) |
| `project` | Project name (persists if project deleted) |
| `user` | Originating user |
| `sender` | Process identifier — for self-recognition + filtering |
| `depends_on` | Parent event id for chains |
| `status` | `pending`, `in_progress`, `finished`, `failed`, `aborted`, `restarted` |
| `progress` | Float %, broadcast-only |

## Built-in topics

- `entity.<type>.created`, `.deleted`, `.updated`, `.status_changed`
- `log.debug`, `log.info`, `log.warning`, `log.error`, `log.success`
- `server.restart_requested`, `settings.changed`
- Addon-private topics — prefix with the addon name (e.g.
  `myaddon.brew_coffee`).

## Dispatching

```python
# Server-side (async):
from ayon_server.events import dispatch_event

await dispatch_event(
    topic="myaddon.job.started",
    project=project_name,
    user=user_name,
    summary={"job_id": jid},
    payload={...},
    store=True,
)
```

REST: `POST {server}/api/events` with the same field names.

## Subscribing

| Context | API |
|---------|-----|
| Server-side addon | `BaseServerAddon.initialize()` → `self.add_event_listener("topic", async_fn)` |
| Client/launcher | `ayon_api` event helpers on the connection object |
| Service | **enroll pattern** (below) |

## The enroll pattern — canonical service loop

`POST /api/enroll` atomically:
1. Finds an unprocessed event matching `sourceTopic` + optional `filter`.
2. Creates a linked `targetTopic` event with `depends_on = source_id`.
3. Returns both ids.

This is how you write an idempotent worker that can safely run in multiple
replicas — each event goes to exactly one of them.

```python
import time, requests
from .config import config

session = requests.session()
session.headers.update({
    "X-Api-Key": config.api_key,
    "Content-Type": "application/json",
})

while True:
    time.sleep(2)
    res = session.post(f"{config.server_url}/api/enroll", json={
        "sourceTopic": "entity.folder.status_changed",
        "targetTopic": "example.brew_coffee_on_approve",
        "sender":      config.service_name,
        "description": "Brewing on approve",
        "filter": {
            "conditions": [
                {"key": "payload/newValue", "value": "Approved"},
            ]
        },
    })
    if res.status_code != 200:
        continue

    j = res.json()
    source_id, target_id = j["dependsOn"], j["id"]
    source = session.get(
        f"{config.server_url}/api/events/{source_id}"
    ).json()

    session.patch(f"{config.server_url}/api/events/{target_id}", json={
        "status": "in_progress", "description": "working",
    })

    # ... do work ...

    session.patch(f"{config.server_url}/api/events/{target_id}", json={
        "status": "finished", "description": "done",
    })
```

Filter `key` uses dotted access: `summary/entity_type`, `payload/newValue`.

## Managing events via REST

- `POST /api/events` — dispatch
- `GET /api/events/{id}` — fetch
- `PATCH /api/events/{id}` — update status / progress / description
- GraphQL `events(topic:..., status:..., project:..., first:...)` for
  queries.

## ASH — Ayon Service Host

Repo: `github.com/ynput/ash`. Python 3.10 supervisor:

- Polls the server's services table.
- Starts each declared service (typically via Docker).
- Injects `AYON_SERVER_URL` + `AYON_API_KEY` into the service env.
- Restarts them on exit.

Docker deployment: mount `/var/run/docker.sock` into ASH and give it a
**fixed hostname** (not a random hash) so the server can identify the host
stably.

## Writing a service

```
ayon-myservice/
└─ services/
   └─ processor/
      ├─ Dockerfile
      ├─ requirements.txt
      └─ service.py
```

`service.py` is a long-lived process that loops on `/api/enroll`. Log to
stdout; ASH captures it.

Declare from the addon's `package.py`:

```python
services = {
    "processor": {
        "image": "ghcr.io/mystudio/ayon-myservice-processor:0.1.0",
        # optional extra env, ports
    }
}
```

Once uploaded + included in a bundle, the server's services page lists it,
admins configure replicas, ASH runs it.

## In-process listener vs external service — rule of thumb

- **HTTP-lifetime reaction, <1s work** → listener on the server addon.
- **>30s work, external calls, rate-limit-sensitive, survives restarts** →
  service with enroll loop.

## Debugging

- GraphQL `events(topic:..., status:...)` is the fastest event audit.
- Always set `sender` to a **stable** identifier so you can track "my
  events" across replicas.
- For WebSocket/fire-and-forget UI hints, Redux devtools shows the
  frontend's WS stream.

## Deeper reading

- `09-events-services.md` — full file with Sources

## Sources

- <https://docs.ayon.dev/docs/dev_event_system>
- `{server}/api` / `{server}/graphiql` — live schema

### Source repos

- <https://github.com/ynput/ash> — Ayon Service Host supervisor (Python 3.10)
- <https://github.com/ynput/ayon-backend/tree/develop/ayon_server/events> — server event system source
- <https://github.com/ynput/ayon-backend/tree/develop/api> — `/api/events`, `/api/enroll` endpoint source
- <https://github.com/ynput/ayon-python-api> — client-side event dispatch + WebSocket helpers
- Example services — <https://github.com/ynput/ayon-ftrack>, <https://github.com/ynput/ayon-shotgrid>, <https://github.com/ynput/ayon-kitsu> (all use the enroll pattern)
