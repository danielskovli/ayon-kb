---
name: ayon-apis
description: Ayon programmatic APIs — `ayon-python-api` / `ayon_api` (connection via `AYON_SERVER_URL`+`AYON_API_KEY`, `ServerAPI`/`ServerAPIBase`, `get_projects`/`get_folders`/`get_tasks`/`get_products`/`get_versions`/`get_representations`, raw HTTP methods); REST (bearer / `X-Api-Key` / `X-as-user`, endpoint groups, pagination); GraphQL (**query-only**, Relay `edges`/`node`/`pageInfo`, `/graphiql`). Use when scripting against Ayon, writing a service, wiring external automation, or debugging API calls.
when_to_use: Triggered by "ayon-python-api", "ayon_api", "get_server_api_connection", "ServerAPI", "REST API", "GraphQL", "X-Api-Key", "Bearer token", "/api/projects", "/api/folders", "/graphiql", "/graphql", "OpenAPI", "Swagger", "WebSocket events", "raw_get", "GraphQlQuery".
---

# Ayon APIs — Python, REST, GraphQL

Ayon exposes three developer APIs:

- **`ayon-python-api`** — canonical Python client (REST + GraphQL + events)
- **REST** — full CRUD + mutations
- **GraphQL** — **read-only** (queries only)
- **WebSocket** — event stream

Plus C++ (`ayon-cpp-api`) and USD resolver (`ayon-usd`).

Interactive docs (live):
- REST Swagger: `{server}/api`
- GraphiQL: `{server}/graphiql`
- OpenAPI JSON: `{server}/openapi.json`
- Public playground: `https://playground.ayon.app/api` + `/explorer`

---

## `ayon-python-api` (`ayon_api`)

### Install + connect

```bash
pip install ayon-python-api
```

```python
import os, ayon_api

os.environ["AYON_SERVER_URL"] = "https://ayon.mystudio.com"
os.environ["AYON_API_KEY"]    = "xxxxxxxxxxxx"

con = ayon_api.get_server_api_connection()
```

Multiple concurrent connections: instantiate `ayon_api.ServerAPIBase`
directly (env-driven singleton is `ServerAPI`).

### Convenience methods (re-exported on the module)

```python
ayon_api.get_projects()
ayon_api.get_project(project_name)

ayon_api.get_folders(project_name, folder_ids=[...], folder_paths=[...])
ayon_api.get_tasks(project_name, folder_ids=[...])
ayon_api.get_products(project_name, folder_ids=[...], product_types=[...])
ayon_api.get_versions(project_name, product_ids=[...])
ayon_api.get_representations(project_name, version_ids=[...])

ayon_api.get_users()
ayon_api.get_events(topics=[...], projects=[...], statuses=[...])

ayon_api.dispatch_event("myaddon.topic", project=p, summary={...})

ayon_api.download_addon_private_file(
    addon_name="houdini", addon_version="0.3.1",
    filename="client.zip", destination_dir="/tmp",
)
```

Exact names/signatures drift between releases — use `dir(ayon_api)` for
the version you target.

### Low-level HTTP (every REST endpoint)

```python
ayon_api.get("projects")                     # kwargs → query params
ayon_api.post("auth/login", name=..., password=...)
ayon_api.patch(f"projects/{p}", attrib={"fps": 24.0})
ayon_api.put(f"users/{u}", name=u, password=..., ...)
ayon_api.delete(f"events/{eid}")
```

`raw_get / raw_post / raw_put / raw_patch / raw_delete` pass through to
`requests` directly (use for streaming, custom headers, timeouts).

### Operations sessions

For batching mutations atomically, `ayon_api` exposes
`create_session()` / `OperationsSession`. API evolves — check current
README.

### Events

```python
ayon_api.dispatch_event("myaddon.topic", project=p, summary={...})
```

WebSocket helpers on the connection object stream live events.

---

## REST API

### Auth

**User token** (via `/api/auth/login`):

```
Authorization: Bearer <token>
```

**Service API key** (from Studio Settings → API Keys):

```
X-Api-Key: <key>
X-as-user: <username>       # optional — impersonation
```

Almost all endpoints require auth (even when not explicitly noted).

### Endpoint groups

Verify on `{server}/api`. Canonical groupings:

- `/auth` — login, session, me
- `/projects` — CRUD; nested folders/tasks/products/versions/representations; attribs
- `/events` — dispatch, get, update, delete
- `/enroll` — service enrollment (see `ayon-events-services`)
- `/addons` — list, upload, settings, and custom endpoints:
  `/addons/{name}/{version}/{endpoint}`
- `/users`, `/accessGroups`, `/roles`
- `/attributes` — custom attribute management
- `/system` — health, restart
- `/desktop`, `/services`, `/bundles`, `/installers` — deployment

### Login + list example

```bash
TOKEN=$(curl -s -X POST "$URL/api/auth/login" \
  -H 'Content-Type: application/json' \
  -d '{"name":"admin","password":"secret"}' | jq -r .token)

curl -s "$URL/api/projects" -H "Authorization: Bearer $TOKEN"
```

### Creating a user (service key)

```python
user = {
    "name": "demouser00",
    "password": "ayon",
    "attrib": {"fullName": "Jane", "email": "j@example.com"},
    "data":   {"defaultAccessGroups": ["artist"]},
}
ayon_api.put(f"users/{user['name']}", **user)
```

---

## GraphQL

**Query-only** — mutations go through REST. Endpoint `{server}/graphql`.
Explorer: `{server}/graphiql`.

### Typical roots

- `projects { edges { node { ... } } }`
- `project(name: "foo") { folders { ... } }`
- Within `project`: `folders`, `tasks`, `products`, `versions`,
  `representations`
- `events(topic:..., status:..., first:50)`
- `users`, `accessGroups`

### Relay pagination

```graphql
query {
  projects(first: 50) {
    edges { cursor node { name } }
    pageInfo { endCursor hasNextPage }
  }
}
```

### Combined query sample

```graphql
query ShotsWithLatestAnim($project: String!) {
  project(name: $project) {
    folders(folderTypes: ["Shot"], first: 500) {
      edges {
        node {
          path
          tasks(first: 10) { edges { node { name taskType } } }
          products(productTypes: ["animation"]) {
            edges {
              node {
                name
                latestVersion { version status }
              }
            }
          }
        }
      }
    }
  }
}
```

### From Python

```python
q = """query { projects { edges { node { name } } } }"""
res = ayon_api.post("graphql", query=q)
```

Helper class: `ayon_api.graphql.GraphQlQuery` (check version).

---

## Headers summary

| Header | Use |
|--------|-----|
| `Authorization: Bearer <token>` | user session |
| `X-Api-Key: <key>` | service account |
| `X-as-user: <username>` | impersonation (service-key only) |
| `X-Sender: <id>` | optional; dispatcher ID; events filter self by it |
| `Content-Type: application/json` | for POST / PUT / PATCH |

## Deeper reading

- `10-apis.md` — full file with Sources

## Sources

- <https://docs.ayon.dev/docs/dev_api_python>
- <https://docs.ayon.dev/docs/dev_api_rest>
- <https://docs.ayon.dev/docs/dev_api_graphql>
- <https://docs.ayon.dev/ayon-python-api/>

### Source repos

- <https://github.com/ynput/ayon-python-api> — `ServerAPI`, `ServerAPIBase`, convenience methods
- <https://github.com/ynput/ayon-python-api/tree/main/ayon_api> — `graphql.py` (`GraphQlQuery`), entity hub, operations session source
- <https://github.com/ynput/ayon-backend/tree/develop/api> — REST endpoint implementations
- <https://github.com/ynput/ayon-backend/tree/develop/ayon_server/graphql> — GraphQL schema / resolvers
- <https://github.com/ynput/ayon-cpp-api> — C++ client
- <https://github.com/ynput/ayon-usd> — USD resolver (can be invoked from the same API surface)
