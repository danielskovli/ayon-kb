# 09 — Events & Services

## Two kinds of events

| Flavour | Stored? | Delivery | Use |
|---------|---------|----------|-----|
| **Persistent** | Postgres | WebSocket fan-out + poll | jobs, audit, depends-on chains |
| **Fire-and-forget** | No | WebSocket only | UI hints, alerts, "reload this view" |

Pass `store=False` at dispatch time to make an event fire-and-forget.

## Event fields

| Field | Notes |
|-------|-------|
| `topic` | required; e.g. `entity.folder.created` |
| `id` | auto UUID |
| `hash` | optional — prevents duplicate dispatch from same source |
| `summary` | small JSON for clients (the "headline") |
| `payload` | full JSON, REST-only (not broadcast over WS) |
| `project` | project name (persists if project deleted) |
| `user` | originating user (persists if user deleted) |
| `sender` | process identifier for self-recognition |
| `depends_on` | parent event id for chains |
| `status` | `pending`, `in_progress`, `finished`, `failed`, `aborted`, `restarted` |
| `progress` | float % — broadcast-only, not persisted |

## Built-in topic conventions

- `entity.<type>.created`, `entity.<type>.deleted`, `entity.<type>.updated`,
  `entity.<type>.status_changed`
- `log.debug`, `log.info`, `log.warning`, `log.error`, `log.success`
- `server.restart_requested`
- `settings.changed`
- Addon-private topics should be prefixed with the addon name (e.g.
  `myaddon.brew_coffee`).

## Dispatching

- Python (server-side): `ayon_server.events.dispatch_event(...)`.
- REST: `POST /api/events`.

```python
await dispatch_event(
    topic="myaddon.job.started",
    project=project_name,
    user=user_name,
    summary={"job_id": jid},
    payload={...},
    store=True,
)
```

## Subscribing

- **Server-side addon**: `add_event_listener("topic", coroutine)` from
  `BaseServerAddon.initialize()`.
- **Client/launcher addon**: subscribe via `ayon_api` events module.
- **Service**: use the **enroll** pattern.

## The enroll pattern (services)

`POST /api/enroll` atomically:

1. Looks for unprocessed events matching `sourceTopic` (+ filter).
2. Creates a linked `targetTopic` event with `depends_on = source_id`.
3. Returns both ids.

This is the canonical way to write an idempotent worker: many service replicas
can call `/enroll` and each event is handed to exactly one of them.

```python
import time, requests

session = requests.session()
session.headers.update({"X-Api-Key": cfg.api_key, "Content-Type": "application/json"})

while True:
    time.sleep(2)
    res = session.post(f"{cfg.server_url}/api/enroll", json={
        "sourceTopic": "entity.folder.status_changed",
        "targetTopic": "example.brew_coffee_on_approve",
        "sender":      cfg.service_name,
        "description": "Brewing on approve",
        "filter": {
            "conditions": [{"key": "payload/newValue", "value": "Approved"}]
        },
    })
    if res.status_code != 200:
        continue

    j = res.json()
    source_id, target_id = j["dependsOn"], j["id"]
    source = session.get(f"{cfg.server_url}/api/events/{source_id}").json()

    session.patch(f"{cfg.server_url}/api/events/{target_id}", json={
        "status": "in_progress", "description": "working"
    })

    # ... do work ...

    session.patch(f"{cfg.server_url}/api/events/{target_id}", json={
        "status": "finished", "description": "done"
    })
```

Filter keys support dotted access into `summary` and `payload`
(`summary/entity_type`, `payload/newValue`, …).

## Managing events via REST

- `POST /api/events` — dispatch
- `GET /api/events/{id}` — fetch
- `PATCH /api/events/{id}` — update status/progress/description
- Delete / list / query via GraphQL (topics, project, user, status, time
  range).

## Services overview (ASH)

`ash` (github.com/ynput/ash) is a Python 3.10 supervisor:

- Polls the server's services table.
- Starts each declared service (typically via Docker).
- Injects `AYON_SERVER_URL` + `AYON_API_KEY` env.
- Restarts them when they exit.

In a docker deploy, **mount `/var/run/docker.sock`** into the ASH container and
give it a fixed hostname (not a random hash) so the server can stably identify
the host.

### Writing a service

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

### Declaring a service from the addon

In `package.py`:

```python
services = {
    "processor": {
        "image": "ghcr.io/mystudio/ayon-myservice-processor:0.1.0",
        # optional extra env, ports, etc.
    }
}
```

Once uploaded + included in a bundle, the server's services page lists it,
admins can configure replicas, and ASH will run it.

## Debugging tips

- GraphQL `events(topic:..., status:...)` is the easiest way to audit event
  flow.
- When writing a service, always set `sender` to a stable identifier so you
  can track "my events" across replicas.
- For fire-and-forget UI hints, test in the browser console: the frontend's
  WebSocket surface is debuggable in Redux devtools.

## Sources

- <https://docs.ayon.dev/docs/dev_event_system> — persistent vs fire-and-forget, event fields, enroll pattern, full service example
- <https://github.com/ynput/ash> — ASH purpose, docker-sock mount, stable hostname
- `{server}/api` → `/events`, `/enroll` — live schema for dispatch/update/query
- `{server}/graphiql` — `events(topic:..., status:..., first:...)` query root for auditing
