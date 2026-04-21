# 16 — Site sync & colorspace

Two cross-cutting concerns every plugin developer hits sooner or later.

## Site sync

### What is a site?

A **site** is a storage location: a physical disk, network share, cloud
bucket, or SFTP host. Every Ayon install has at least:

- `studio` — the central shared storage (the assumption is "everyone can
  reach it eventually").
- `local` — the workstation the launcher is running on.

A user is always working with **two sites simultaneously**:

- **active site** — where they read/write locally (often `local`, sometimes
  `studio` directly).
- **remote site** — the other side of the sync (often `studio`).

### Per-representation tracking

Each **representation** is independently tracked per site. A file that has
been published to `studio` but not yet synced to `local` will show up with
a download badge in the Loader / Scene Inventory. The Site Sync service
then transfers files in the background.

Two sites can share underlying storage — e.g. `studio` (SMB mount) and
`sftp` (SFTP view of the same bytes). Publishing to one automatically
satisfies the other.

### Providers

| Provider | What it does |
|----------|--------------|
| `local_drive` | Local disks, mounted network drives, external drives |
| `gdrive` | Google Drive (Google API) |
| `sftp` | SFTP with username/password or SSH keys |

Additional providers can be written by studios. "Active" and "remote" are
**roles**, not providers.

### Admin setup

1. Enable the Site Sync addon globally.
2. Assign the required sites to each project in Project Settings.
3. Run the sync service (background worker):
   ```
   ayon syncservice --active_site studio
   ```
   This runs as an ASH service in production.

### Publishing + site sync

- When publishing, the representation is registered on the configured
  remote site.
- A "studio"-published file on a same-storage alternate site becomes
  immediately available there too.
- The publisher can be configured to mark representations as
  `local`-only, `studio`-only, or both — via a Core addon profile.

### Loading + site sync

- The Loader shows a per-representation site widget (local + remote,
  "sync needed" vs "available").
- `download_container()` / Scene Inventory "Download" action enqueues a
  transfer from the remote site to the active site and (normally) blocks
  the load until it finishes.

### Developer hooks

- Check a representation's site availability before acting — use the
  site-sync addon's API rather than guessing from file paths.
- Don't assume the file is where anatomy says it is on the current
  workstation — a `local`-only representation may not yet exist on
  `studio`.
- For custom providers, look at the Site Sync addon source; providers
  implement a small `SyncProvider` interface.

### Known caveats

- The `help.ayon.app` Site Sync article explicitly lists only
  `local_drive`, `gdrive`, `sftp`. Any mention of Dropbox is unofficial.
- Service replicas don't coordinate — run **one active worker per
  site-pair** or queue decisions will fight.

## Colorspace

### Representation-level metadata

Published representations carry a `colorspaceData` entry:

```json
{
  "colorspace": "sRGB - Display",
  "config": {
    "path":     "Z:/studio/ocio/prod.ocio",
    "template": "{root[work]}/ocio/prod.ocio"
  }
}
```

- `colorspace` — string identifier used by loaders and downstream tools.
- `config.path` — absolute, platform-resolved path.
- `config.template` — the unformatted template, carried so a remote
  publisher on a different platform can resolve a working path locally.

### Extractor side

A publish plugin should inherit from
`ColormanagedPyblishPluginMixin` (Ayon's colorspace helper) and call
`set_representation_colorspace(representation, colorspace=...)` before
integration. If the extractor doesn't specify a colorspace, the mixin
applies **file rules** from the OCIO config.

```python
import pyblish.api
from ayon_core.pipeline.colorspace import ColormanagedPyblishPluginMixin


class ExtractBeauty(pyblish.api.InstancePlugin, ColormanagedPyblishPluginMixin):
    order = pyblish.api.ExtractorOrder

    def process(self, instance):
        ...  # render the file
        repre = {"name": "exr", "files": [...], ...}
        self.set_representation_colorspace(repre, colorspace="ACES - ACEScg")
        instance.data.setdefault("representations", []).append(repre)
```

### Loader side

Loaders read `colorspaceData` from the representation entity and apply the
colorspace in the DCC (e.g. Maya `colorManagementPrefs`, Nuke `colorspace`
knob). Loaders should tolerate **config mismatches** between publish-time
and load-time: prefer the colorspace **name** over the bytes of the file's
embedded metadata.

### Host-side integration

For a DCC integration to participate in color management:

1. Add the **`imageio`** sub-schema to the host's settings (`ayon-core`
   ships a reusable `IMAGEIO_SETTINGS` schema).
2. Either set the OCIO env var (`OCIO=...`) from a launch hook **or** call
   the host's API to set the config path.
3. Use `ayon_core.pipeline.colorspace.get_imageio_config(...)` to resolve
   the per-project OCIO path.

### Distribution of configs

The OCIO config can ship as a **public static file** inside the addon
(`public/ocio/my_studio.ocio`), or be pulled from a studio-managed share.
The FAQ / Core settings references a default Ayon-distributed preset when
none is configured.

## Sources

- <https://help.ayon.app/en/articles/3882937-site-sync> — sites, providers, active/remote roles, `syncservice`
- <https://help.ayon.app/articles/3005275-core-addon-settings> — Extract Review / transcoding profiles
- <https://docs.ayon.dev/docs/dev_colorspace> — `colorspaceData`, `ColormanagedPyblishPluginMixin.set_representation_colorspace`, `get_imageio_config`, file-rules fallback
- `ayon_core.pipeline.colorspace` — in-code helpers (check `ayon-core` for the current API surface)
