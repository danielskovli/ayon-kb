# 10 — APIs (Python, REST, GraphQL)

Ayon exposes three developer APIs:

- **`ayon-python-api`** — canonical Python client (REST + GraphQL + events).
- **REST** — full CRUD + mutations.
- **GraphQL** — **read-only** (queries only). Useful for combining reads.
- **WebSocket** — event stream (consumed by `ayon-python-api`, `ayon-frontend`).

There is also a **C++ client** (`ayon-cpp-api`) and a **USD resolver**
(`ayon-usd`), both with their own docs.

## Where to find the interactive docs

- REST Swagger: `{server}/api` (or `https://playground.ayon.app/api`)
- GraphQL explorer: `{server}/graphiql`
  (or `https://playground.ayon.app/explorer`)

## `ayon-python-api` (`ayon_api`)

### Install & connect

```bash
pip install ayon-python-api
```

```python
import os
import ayon_api

os.environ["AYON_SERVER_URL"] = "https://ayon.mystudio.com"
os.environ["AYON_API_KEY"]    = "xxxxxxxxxxxx"

con = ayon_api.get_server_api_connection()
```

For multiple simultaneous connections, instantiate `ayon_api.ServerAPIBase`
directly.

### Convenience methods

`ayon_api` re-exports common methods on the module for ergonomics:

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
```

Exact names / signatures drift between releases — `dir(ayon_api)` on the
version you target is the source of truth.

### Low-level HTTP

Every REST endpoint is reachable via:

```python
ayon_api.get("projects")                         # query params via kwargs
ayon_api.post("auth/login", name=..., password=...)
ayon_api.patch(f"projects/{p}", attrib={"fps": 24.0})
ayon_api.put(f"users/{u}", name=u, password=..., ...)
ayon_api.delete(f"events/{eid}")
```

`raw_get / raw_post / raw_put / raw_patch / raw_delete` bypass kwarg massaging
and pass through to `requests` directly (use when you need control over
headers, streaming, timeouts).

### Operations sessions

For batching mutations atomically `ayon_api` exposes an **operations session**
(`create_session()` / `OperationsSession`). Check the current version's docs —
the API has been evolving. When present, it lets you queue up creates/updates
and commit them in one request.

### Addon file download

```python
ayon_api.download_addon_private_file(
    addon_name="houdini",
    addon_version="0.3.1",
    filename="client.zip",
    destination_dir="/tmp",
)
```

### Events

```python
ayon_api.dispatch_event("myaddon.topic", project=p, summary={...})
```

The Python client also exposes WebSocket helpers for streaming live events.

## REST API

### Auth

Two modes:

**User token** (from `/api/auth/login`):

```
Authorization: Bearer <token>
```

**Service API key** (from Studio Settings → API Keys):

```
X-Api-Key: <key>
X-as-user: <username>   # optional — impersonate
```

Almost all endpoints require auth; the docs call this out — "even if it's not
explicitly mentioned."

### Endpoint shape

Everything is `{server}/api/...`. Canonical groupings (verify on
`{server}/api`):

- `/auth` — login, session, me
- `/projects` — CRUD projects; attribs; folder/task/product/version/representation
  nested CRUD
- `/events` — dispatch, get, update, delete
- `/enroll` — service enrollment
- `/addons` — list, upload, settings, custom endpoints:
  `/addons/{name}/{version}/{endpoint}`
- `/users`, `/accessGroups`, `/roles`
- `/attributes` — custom attribute management
- `/system` — health, restart
- `/desktop`, `/services`, `/bundles`, `/installers` — deployment

Responses are JSON. List endpoints usually paginate (`cursor` + `first`).

### Example — login + list projects

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

## GraphQL

**Query-only** — use REST for mutations. Endpoint: `{server}/graphql`.
Explorer UI: `{server}/graphiql`.

### Typical roots

- `projects { edges { node { ... } } }`
- `project(name:"foo") { folders { edges { node { ... } } } }`
- `folders`, `tasks`, `products`, `versions`, `representations` (within project)
- `events(topic:..., status:..., first:50)`
- `users`, `accessGroups`

### Pagination

Relay-style cursors on list fields:

```graphql
query {
  projects(first: 50) {
    edges { cursor node { name } }
    pageInfo { endCursor hasNextPage }
  }
}
```

### Sample

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

GraphQL fits nicely with `ayon_api.GraphQlQuery` helper classes — see
`ayon_api.graphql`. For ad-hoc queries, POST the query string to `/graphql`:

```python
q = """query { projects { edges { node { name } } } }"""
res = ayon_api.post("graphql", query=q)
```

## WebSocket events

`ayon-python-api` exposes event subscription via its connection object. In
browser/JS land the frontend uses a WS manager that dispatches to Redux. Topic
filters are server-side.

## Headers summary

| Header | Use |
|--------|-----|
| `Authorization: Bearer ...` | user session |
| `X-Api-Key: ...` | service account |
| `X-as-user: ...` | impersonation (service key only) |
| `X-Sender: ...` | optional; identifies dispatcher; events filter self by it |
| `Content-Type: application/json` | for POST/PUT/PATCH |

## Sources

- <https://docs.ayon.dev/docs/dev_api_python> — install, `get_server_api_connection`, high-level + raw methods, example put
- <https://docs.ayon.dev/docs/dev_api_rest> — bearer + `X-Api-Key` / `X-as-user`, login flow
- <https://docs.ayon.dev/docs/dev_api_graphql> — query-only, `/graphiql`, Relay edges pattern
- <https://github.com/ynput/ayon-python-api> — README (connection pattern, installation, TODOs)
- <https://docs.ayon.dev/ayon-python-api/> — full Python API reference (method-by-method)
- `{server}/api` — Swagger schema (authoritative for REST endpoints)
- `{server}/graphiql` — schema explorer (authoritative for GraphQL roots and pagination)
- `{server}/openapi.json` — raw OpenAPI for code generation
