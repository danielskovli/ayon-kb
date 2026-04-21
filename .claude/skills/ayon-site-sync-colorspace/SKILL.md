---
name: ayon-site-sync-colorspace
description: Ayon cross-cutting concerns — Site Sync (sites as storage locations, active/remote roles, providers `local_drive`/`gdrive`/`sftp`, per-representation tracking, `syncservice`) and colorspace / OCIO (`ColormanagedPyblishPluginMixin.set_representation_colorspace`, `colorspaceData` on representations, `get_imageio_config`, per-host `imageio` settings). Use when writing multi-site-aware plugins, integrating OCIO into a host, embedding colorspace in extractors, or debugging "file not found locally" / wrong colorspace on load.
when_to_use: Triggered by "site sync", "sites", "active site", "remote site", "gdrive", "SFTP provider", "syncservice", "representation site availability", "OCIO", "colorspace", "ColormanagedPyblishPluginMixin", "set_representation_colorspace", "colorspaceData", "imageio", "ACEScg", "display transform".
---

# Site Sync & Colorspace

Two cross-cutting concerns every plugin author eventually hits.

---

## Site Sync

### What is a site?

A **site** is a storage location — physical disk, network share, cloud
bucket, SFTP host. Every Ayon install has at least:

- **`studio`** — central shared storage (the "everyone can eventually
  reach it" assumption)
- **`local`** — the workstation the launcher runs on

A user always works with **two sites simultaneously**:

- **active site** — where they read/write locally (often `local`)
- **remote site** — the sync counterpart (often `studio`)

### Per-representation tracking

Each representation is independently tracked per site. A file published
to `studio` but not yet synced to `local` shows a download badge in the
Loader / Scene Inventory. Two sites can share underlying storage
(e.g. `studio` + `sftp` view of the same bytes) — publishing to one
satisfies both.

### Providers

| Provider | What it does |
|----------|--------------|
| `local_drive` | Local disks, network mounts, externals |
| `gdrive` | Google Drive (Google API) |
| `sftp` | SFTP with username/password or SSH keys |

Studios can write custom providers. "Active" and "remote" are **roles**,
not providers.

### Admin flow

1. Enable the Site Sync addon globally.
2. Assign required sites per project.
3. Run the background sync worker:
   ```
   ayon syncservice --active_site studio
   ```
   In production this is an ASH service.

### Publishing + sync

- Publishing registers representations on the configured remote site.
- Same-storage alternates auto-satisfy on publish.
- Core addon profiles can restrict a representation to `local`-only,
  `studio`-only, or both.

### Loading + sync

- The Loader shows a per-representation site widget (local / remote,
  "sync needed" / "available").
- Scene Inventory **Download** enqueues a transfer from remote → active
  and normally blocks the load until the transfer completes.

### Developer notes

- **Never assume the file is where anatomy says it is** on this
  workstation. A `local`-only representation may not exist on `studio`
  yet. Check availability via the Site Sync addon's API before acting.
- For custom providers, inherit the `SyncProvider` interface in the
  site-sync addon source.
- Run **one active worker per site-pair** — replicas don't coordinate
  and will fight over decisions.

### Known caveats

- The help article lists only `local_drive`, `gdrive`, `sftp`. Any
  mention of Dropbox is unofficial/outdated.

---

## Colorspace (OCIO)

### `colorspaceData` on representations

Published representations carry:

```json
{
  "colorspace": "sRGB - Display",
  "config": {
    "path":     "Z:/studio/ocio/prod.ocio",
    "template": "{root[work]}/ocio/prod.ocio"
  }
}
```

- `colorspace` — string ID used by loaders and downstream tools
- `config.path` — absolute platform-resolved path
- `config.template` — unformatted template, carried so a remote publisher
  on a different platform can re-resolve locally

### Extractor side

Inherit `ColormanagedPyblishPluginMixin`, call
`set_representation_colorspace(repre, colorspace=...)` before integration.
If no colorspace is specified, the mixin applies **file rules** from the
OCIO config.

```python
import pyblish.api
from ayon_core.pipeline.colorspace import ColormanagedPyblishPluginMixin


class ExtractBeauty(pyblish.api.InstancePlugin,
                    ColormanagedPyblishPluginMixin):
    order = pyblish.api.ExtractorOrder

    def process(self, instance):
        ...   # render the file
        repre = {"name": "exr", "files": [...], ...}
        self.set_representation_colorspace(repre, colorspace="ACES - ACEScg")
        instance.data.setdefault("representations", []).append(repre)
```

### Loader side

Read `colorspaceData` from the representation entity and apply the
colorspace in the DCC (e.g. Maya `colorManagementPrefs`, Nuke `colorspace`
knob). Prefer the colorspace **name** over file-embedded metadata —
publish-time and load-time OCIO configs may differ.

### Host-side integration

To participate in color management, a host needs to:

1. Add the **`imageio`** sub-schema to its settings (`ayon-core` ships a
   reusable `IMAGEIO_SETTINGS` schema).
2. Either set the `OCIO` env var from a launch hook, or call the host's
   native API to set the config path.
3. Use `ayon_core.pipeline.colorspace.get_imageio_config(...)` to
   resolve the per-project OCIO path.

### Distribution of OCIO configs

- Ship as a **public static file** inside the addon (`public/ocio/...`)
- Or pull from a studio-managed network share
- Ayon ships default presets used when none is configured.

## Deeper reading

- `16-site-sync-colorspace.md` — full file with Sources
- Skill `ayon-publishing` — extractor/mixin context
- Skill `ayon-loaders-inventory` — loader perspective

## Sources

- <https://help.ayon.app/en/articles/3882937-site-sync>
- <https://docs.ayon.dev/docs/dev_colorspace>
- <https://help.ayon.app/articles/3005275-core-addon-settings>

### Source repos

- <https://github.com/ynput/ayon-sitesync> — Site Sync addon (providers, syncservice, SyncProvider interface)
- <https://github.com/ynput/ayon-core/blob/develop/client/ayon_core/pipeline/colorspace.py> — `ColormanagedPyblishPluginMixin`, `set_representation_colorspace`, `get_imageio_config`
- <https://github.com/ynput/ayon-core/tree/develop/client/ayon_core/plugins/publish> — `extract_*_transcode` / `extract_review` reference plugins that use colorspace metadata
- <https://github.com/ynput/ayon-usd> — USD resolver + pipeline (colorspace overlaps with USD material output)
