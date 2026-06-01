# Architecture patterns

## File organisation

All integrations live under `custom_components/<domain>/`. The full possible
layout (legend: **required**, *conditional*, _optional_):

```
custom_components/<domain>/
├── __init__.py              # required — async_setup_entry / async_unload_entry / async_reload_entry only
├── const.py                 # required — DOMAIN, LOGGER, ATTRIBUTION, defaults
├── data.py                  # required — TypedDicts, dataclass, ConfigEntry type alias
├── entity.py                # required — base entity (CoordinatorEntity subclass)
├── coordinator.py           # required — DataUpdateCoordinator[PayloadType]
├── <purpose>_coordinator.py # *conditional — when polling cadences differ (e.g. map_coordinator.py)
├── api.py                   # required — SDK wrapper / HTTP client
│                            #   OR api/ package (client.py + errors.py + responses.py + ...)
├── <subsystem>/             # *conditional — auxiliary SDK-like subpackages
│                            #   (e.g. cloud/, bluetooth/) with own client + errors + types
├── config_flow.py           # required — user, reauth, reauth_confirm, reconfigure steps
├── options_flow.py          # *conditional — only when there are user-tunable options
├── diagnostics.py           # *conditional — required for Platinum quality scale
├── repairs.py               # *conditional — only when registering repair issues
├── quality_scale.yaml       # *conditional — required once you declare a Quality Scale tier
├── icons.json               # _optional — custom MDI icons per entity
├── manifest.json            # required
├── exceptions/              # *conditional — central exception package:
│   ├── __init__.py          #   re-exports
│   ├── api_client_error.py  #   one file per exception
│   ├── api_client_authentication_error.py
│   └── api_client_communication_error.py
│                            #   OR per-subsystem errors files (api/errors.py, cloud/errors.py)
│                            #   when the integration has multiple subpackages
├── translations/
│   ├── en.json              # required
│   └── <locale>.json        # _optional — one per additional locale
├── brand/                   # required — icon.png, logo.png (+ @2x), icon.svg — always shipped in-repo
└── <platform>/              # *conditional — one directory per platform when >1 entity
    ├── __init__.py          # async_setup_entry only
    └── <entity_name>.py     # one file per entity class
```

When a platform has a single entity (e.g. `switch.py`, `number.py`), a flat
file is fine. When it has multiple entities (e.g. `sensor/temperature.py`,
`sensor/battery.py`), group them into a package directory with `__init__.py`
containing only `async_setup_entry`.

## Layout variants

The skeleton above lists **all** files an integration might have — most repos
do not have every one. The variants worth knowing:

- **`api.py` vs `api/` package.** A flat `api.py` works when the client is
  one class with a handful of helpers. Split into `api/client.py` +
  `api/errors.py` (+ `api/responses.py`, ...) the moment there are multiple
  top-level classes — same "one class per file" rule that applies to
  platforms.
- **Auxiliary `<subsystem>/` packages.** Integrations that talk to more than
  one protocol or cloud (e.g. local LAN + vendor cloud for map decoding)
  isolate each in its own subpackage with its own `client.py` and
  `errors.py`. Keeps blast radius small per subsystem.
- **Central `exceptions/` vs per-subsystem errors files.** Both layouts are
  valid:
  - Central: every error lives in `exceptions/<error>.py`, re-exported from
    `exceptions/__init__.py`. Good for single-API integrations.
  - Per-subsystem: each subpackage owns its own `errors.py`
    (`api/errors.py`, `cloud/errors.py`). Good when subsystems have
    genuinely distinct failure modes.
  Pick one per integration and stick with it.

## One class per file

Every file contains one top-level class. TypedDicts and `type` aliases are
exceptions — they group into `data.py`. Helper functions may live alongside
the single class that uses them (e.g. `_verify_response_or_raise` in `api.py`).

## Runtime data

All integration state lives on `entry.runtime_data: <Domain>Data`, a
`@dataclass` defined in `data.py`. Never use `hass.data[DOMAIN]`. The
`runtime_data` is automatically discarded on unload.

```python
type <Domain>ConfigEntry = ConfigEntry[<Domain>Data]

@dataclass
class <Domain>Data:
    client: <Domain>ApiClient
    coordinator: <Domain>DataUpdateCoordinator
    integration: Integration
```

## Coordinator

- Typed as `DataUpdateCoordinator[PayloadType]` where PayloadType is a
  TypedDict or dataclass.
- `_async_update_data()` returns the typed payload.
- Use `await coordinator.async_config_entry_first_refresh()` during setup
  (not `async_refresh()`). A failed first refresh raises `ConfigEntryNotReady`.
- Pass `always_update=False` when the payload compares cleanly with `__eq__`.
- Error mapping inside `_async_update_data`:
  - Authentication errors → `raise ConfigEntryAuthFailed(...)` (starts reauth)
  - Communication errors → `raise UpdateFailed(...)` (HA retries), unless the
    coordinator implements an explicit, documented degradation strategy
    (grace-period / last-known-data) for a known-flaky upstream — then it may
    return the previous payload instead of raising.

### Multiple coordinators

Use more than one coordinator when different parts of the device need
**different polling cadences** (e.g. fast state poll + slow / on-demand
heavy fetch like a cleaning map). Each one:

- Lives in its own file: `coordinator.py` for the primary one,
  `<purpose>_coordinator.py` (e.g. `map_coordinator.py`) for the others.
- Has its own `PayloadType` and its own `update_interval`.
- Is held in `runtime_data` as a separate field
  (`coordinator`, `map_coordinator`, ...).
- Entities subscribe to whichever coordinator drives their state — never
  conflate two coordinators behind one entity.

Do not split coordinators by entity or by platform — split only when the
underlying data has a different freshness need.

## Config flow

All integrations share a four-step pattern with one `_validate` helper and one
`_credentials_schema` builder:

1. `async_step_user()` — initial setup
2. `async_step_reauth()` → `async_step_reauth_confirm()` — rotate credentials
3. `async_step_reconfigure()` — edit via three-dot menu
4. `async_get_options_flow()` — returns the options flow class

`unique_id` is set from a stable identifier (e.g. slugified username,
device_id). Aborts on duplicate.

## Options flow

One class per file (`options_flow.py`). Changes trigger
`async_reload_entry()` so the coordinator re-instantiates with new
`update_interval`.

## Entity design

**One class per entity.** Never share a generic class parameterised by an
`EntityDescription` subclass with callable fields like `value_fn` or
`action_fn`. Encode behaviour directly via `@property` and class-level
`_attr_*` constants.

Naming: `<Domain><Name><Platform>` — e.g. `<Domain>TemperatureSensor`,
`<Domain>DoorBinarySensor`, `<Domain>BatterySensor`.

Base entity always sets:
- `_attr_attribution = ATTRIBUTION`
- `_attr_has_entity_name = True`
- `device_info` as a `@property`

## Exception hierarchy

```
<Domain>ApiClientError (base)
  ├── <Domain>ApiClientAuthenticationError
  └── <Domain>ApiClientCommunicationError
```

One file per exception class — either centrally under `exceptions/` (re-exported
from `exceptions/__init__.py`) or per-subsystem at `<subsystem>/errors.py`
(see "Layout variants" above). The API client wraps upstream SDK/library
exceptions at the boundary — nothing above catches raw upstream exceptions.

## Translations

Ship at least `en.json` under `translations/`; add other locales as needed. A
parametrised test (`test_translations.py`) verifies that every locale's nested
key set is identical to `en.json`.

Structure:
- `config.step.<step_id>` — flow UI
- `config.error.<reason>` — flow errors
- `config.abort.<reason>` — flow aborts
- `options.step.init` — options flow
- `entity.<platform>.<key>.name` — entity names
- `entity.<platform>.<key>.state` — enum state labels
- `issues.<issue_id>` — repair flow strings

## Diagnostics

`async_get_config_entry_diagnostics` returns a typed payload.
Sensitive keys go into `TO_REDACT: frozenset[str]`.

## HACS requirements

- One integration per repository at `custom_components/<domain>/`.
- `manifest.json` with `domain`, `name`, `version`, `documentation`,
  `issue_tracker`, `codeowners`. `version` is mandatory (SemVer).
- `hacs.json` at repo root pins minimum HA version.
- **Brand assets live in `custom_components/<domain>/brand/` inside the
  repo.** Ship `icon.png`, `logo.png` (+ `@2x` variants), and `icon.svg`
  there. Do **not** rely on the upstream `home-assistant/brands` repo as
  the source — this skill's integrations bundle their own assets so HACS
  installs render correctly without an external dependency.
- Release-please tags releases on merge to main.
