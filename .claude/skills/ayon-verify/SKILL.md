---
name: ayon-verify
description: Verify a claimed Ayon class, method, attribute, or kwarg against the canonical live source on GitHub (`ayon-core`, `ayon-backend`, `ayon-python-api`, or per-DCC repos). Reports whether the symbol exists, its current signature/attributes, and flags renames/moves. Use before committing code that relies on any specific Ayon API surface — especially anything from older docs or memory.
when_to_use: User types `/ayon-verify <symbol>` or asks to "confirm this method exists" / "is this still the right class name" / "verify BaseServerAddon has X" before relying on the KB.
argument-hint: [Class.method or import.path or free text]
disable-model-invocation: true
allowed-tools: WebFetch WebSearch Bash(gh *) Grep
---

# Verify Ayon API symbol against live source

Target: **$ARGUMENTS**

Follow this playbook. Be terse; report findings only.

## Step 1 — pick the right repo

Decide from the target name which repo most likely owns it. Default mapping:

| Symbol shape | Repo |
|--------------|------|
| `ayon_server.*`, `BaseServerAddon`, `SettingsField`, `ActionExecutor` | `ynput/ayon-backend` |
| `ayon_core.*`, `AYONAddon`, `HostBase`, `LoaderPlugin`, `CreateContext`, `Creator`, `OpenPypePyblishPluginMixin`, `PublishValidationError`, `InventoryAction`, `Anatomy`, `ColormanagedPyblishPluginMixin`, `install_host`, `get_current_context` | `ynput/ayon-core` |
| `ayon_api.*`, `ServerAPI`, `ServerAPIBase`, `GraphQlQuery` | `ynput/ayon-python-api` |
| `ayon_applications.*`, `PreLaunchHook`, `Application`, `ApplicationManager` | `ynput/ayon-applications` |
| DCC-specific (`MayaHost`, `NukeHost`, …) | `ynput/ayon-<host>` |
| `ash`, service supervisor | `ynput/ash` |
| `ayon-addon-template` | `ynput/ayon-addon-template` |

If unclear, run `gh search code "symbol_name" --owner ynput` to find it.

## Step 2 — locate the definition

Prefer `gh` + `Grep` over web fetching (faster + accurate):

```bash
gh api repos/ynput/<repo>/contents/<likely-path> --jq '.[].name'
```

Or a direct content fetch on the most likely path. Common paths:

- `ayon-core` client classes: `client/ayon_core/<submodule>/**/*.py`
- `ayon-backend` server classes: `ayon_server/**/*.py`
- `ayon-python-api`: `ayon_api/*.py`

Use `Grep` on a locally-cloned repo if available. Otherwise use the GitHub
search API:

```bash
gh search code "class <ClassName>" --owner ynput --language python
gh search code "def <method_name>" --owner ynput --language python
```

For a specific repo:

```bash
gh api "search/code?q=<symbol>+repo:ynput/ayon-core" --jq '.items[].path'
```

## Step 3 — report

Deliver a concise report with these sections, in this order:

**Status**: `CONFIRMED` | `RENAMED` | `MOVED` | `NOT FOUND` | `DEPRECATED`

**Repo + path**: `ynput/<repo>` · `<file path>` · optional line number

**Signature / surface** (copy-paste from source):
```python
class ClassName(...):
    attr: type = default
    def method(self, arg1, arg2=None, *, kwarg=...) -> ReturnType: ...
```

**Notes**: anything relevant — default values, recently added, deprecation
warnings in docstrings, related classes.

**Confidence**: `HIGH` (found in source), `MEDIUM` (found in docs / README
only), `LOW` (couldn't confirm).

## Step 4 — update the KB if drift was detected

If the live signature doesn't match what the KB claims in `01`–`18` or
`99-open-questions.md`:

1. Note the discrepancy in the report.
2. Offer to update the relevant KB file in place (and its Sources block
   if needed). **Don't silently rewrite** — confirm with the user first.

## Reference — common symbols and their homes

| Symbol | Repo / module |
|---|---|
| `BaseServerAddon` | `ayon-backend` / `ayon_server/addons/base.py` |
| `SettingsField`, `BaseSettingsModel` | `ayon-backend` / `ayon_server/settings/` |
| `ActionExecutor`, `SimpleActionManifest` | `ayon-backend` / `ayon_server/actions/` |
| `dispatch_event`, `EventModel` | `ayon-backend` / `ayon_server/events/` |
| `AYONAddon`, `ITrayAddon`, `IPluginPaths`, `IHostAddon`, `click_wrap` | `ayon-core` / `client/ayon_core/addon/` |
| `HostBase`, `IWorkfileHost`, `ILoadHost`, `IPublishHost` | `ayon-core` / `client/ayon_core/host/` |
| `install_host`, `registered_host`, `AVALON_CONTAINER_ID` | `ayon-core` / `client/ayon_core/pipeline/__init__.py` |
| `CreateContext`, `CreatedInstance`, `BaseCreator`, `Creator`, `HiddenCreator`, `AutoCreator` | `ayon-core` / `client/ayon_core/pipeline/create/` |
| `LoaderPlugin`, `ProductLoaderPlugin`, `LoaderHookPlugin` | `ayon-core` / `client/ayon_core/pipeline/load/plugins.py` |
| `load_container`, `update_container`, `switch_container`, `get_representation_path` | `ayon-core` / `client/ayon_core/pipeline/load/utils.py` |
| `InventoryAction` | `ayon-core` / `client/ayon_core/pipeline/` |
| `OpenPypePyblishPluginMixin` | `ayon-core` / `client/ayon_core/pipeline/publish/` |
| `PublishValidationError`, `PublishXmlValidationError`, `KnownPublishError` | `ayon-core` / `client/ayon_core/pipeline/publish/` |
| `ColormanagedPyblishPluginMixin`, `set_representation_colorspace`, `get_imageio_config` | `ayon-core` / `client/ayon_core/pipeline/colorspace.py` |
| `Anatomy` | `ayon-core` / `client/ayon_core/pipeline/anatomy/` |
| `PreLaunchHook`, `PostLaunchHook`, `Application`, `ApplicationManager` | `ayon-applications` |
| `ServerAPI`, `ServerAPIBase`, `GraphQlQuery` | `ayon-python-api` / `ayon_api/` |
| `BoolDef`, `NumberDef`, `EnumDef`, `TextDef`, etc. | `ayon-core` / `client/ayon_core/lib/attribute_definitions.py` |

These paths drift; treat them as starting hints, not truth.
