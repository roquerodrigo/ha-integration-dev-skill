# Architecture patterns

## File organisation

All integrations live under `custom_components/<domain>/`. The full possible
layout (legend: **required**, *conditional*, _optional_):

```
custom_components/<domain>/
├── __init__.py              # required — async_setup_entry / async_unload_entry / async_reload_entry only
├── const.py                 # required — DOMAIN, LOGGER, ATTRIBUTION, defaults
├── data/                    # required — one TypedDict/dataclass per file, type aliases in __init__.py
│   ├── __init__.py          #   re-exports + type aliases (ConfigEntry, JsonPrimitive, etc.)
│   └── <type_name>.py       #   one file per TypedDict or dataclass
├── entity.py                # required — base entity (CoordinatorEntity subclass when polling)
├── coordinator.py           # standard for polling — DataUpdateCoordinator[PayloadType]
├── <purpose>_coordinator.py # *conditional — when polling cadences differ (e.g. map_coordinator.py)
├── api.py                   # standard for polling — SDK wrapper / HTTP client
│                            #   OR api/ package (client.py + errors.py + responses.py + ...)
├── <subsystem>/             # *conditional — auxiliary SDK-like subpackages
│                            #   (e.g. cloud/, bluetooth/) with own client + errors + types
├── config_flow.py           # required — user, reauth, reauth_confirm, reconfigure steps
├── options_flow.py          # *conditional — only when there are user-tunable options
├── diagnostics.py           # *conditional — add when the entry holds data worth dumping/redacting
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

Every file contains one top-level class — this includes TypedDicts and
dataclasses, which each get their own file under `data/` (a package with an
`__init__.py` re-exporting the public symbols). `type` aliases live in
`data/__init__.py` alongside the re-exports. Helper functions may live
alongside the single class that uses them (e.g. `_verify_response_or_raise`
in `api.py`).

This is a **blueprint invariant**, like `unique_id`-as-property: a legacy
repo carrying a flat multi-class `data.py` is drift to migrate when you touch
that area — not a repo-style override, even if the repo's own `CODE_STYLE.md`
predates the rule.

Two deliberate relaxations (see coding-conventions.md): a TypedDict or `type`
alias with a single consumer may live in that consuming module, and leaf
dataclasses decomposing one payload may share a module — don't force dozens
of micro-files for one response shape.

## Runtime data

All integration state lives on `entry.runtime_data: <Domain>Data`, a
`@dataclass` defined in `data/runtime.py` (re-exported from
`data/__init__.py`). Never use `hass.data[DOMAIN]`. The
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
  **Local-push variant:** when the transport is slow or intermittent by nature
  (e.g. passive BLE through proxies) and the integration has a documented
  degradation strategy, a non-blocking
  `entry.async_create_task(hass, coordinator.async_refresh())` is the accepted
  alternative — blocking setup on a device that is legitimately out of reach
  would keep the entry from ever loading.
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

## Push and local-transport variants

The coordinator/api/entity trio above is the **standard shape for polling
integrations** — it is not mandatory when data arrives by push or the device
cannot report state at all. Three recurring variants, all legitimate:

- **Passive BLE (advertisement-driven).** State arrives through HA's
  Bluetooth advertisement feed (possibly relayed by ESPHome/Shelly proxies),
  not through requests the integration initiates. Connections are slow and
  intermittent, so a blocking first refresh is wrong here — see the
  local-push note under "Coordinator" and document the degradation strategy
  (last-known state, availability windows).
- **MQTT / cloud push with a supervisor.** A long-lived listener task
  receives events and pushes them into entities. Run it via
  `entry.async_create_background_task(hass, coro, name)` — never a bare
  `asyncio.create_task` — so HA cancels it on unload and it doesn't block
  startup. Pair it with an explicit reconnect/backoff supervisor and a
  documented behaviour for the connection-lost window.
- **Assumed state (one-way transports such as IR).** The device cannot
  report state; entities set `_attr_assumed_state = True` and keep an
  optimistic local state updated on each command sent. There may be no
  coordinator and no API client at all — a thin encoder plus the transport
  service is the whole integration. Since the entity state is mutable and
  locally owned, mutable `_attr_*` assigned in `__init__` is the accepted
  form here (see coding-conventions.md).

What stays mandatory in every variant: config flow, typed payloads,
`entry.runtime_data`, translations, and the error-mapping boundary.

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
`EntityDescription` subclass with `value_fn`-style callables. Naming:
`<Domain><Name><Platform>` — e.g. `<Domain>TemperatureSensor`,
`<Domain>DoorBinarySensor`.

Base entity always sets `_attr_has_entity_name = True` and `device_info` as a
`@property`; it also sets `_attr_attribution = ATTRIBUTION` **when the
integration has a third-party data source to credit** (a local-device
integration has nothing to attribute). Member order, property-vs-`__init__`,
the `unique_id` invariant and its push/optimistic exception, and entity
categories are defined in [coding-conventions.md](./coding-conventions.md).

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
  repo, and that directory is what Home Assistant serves.** Since **HA
  2026.3** the core `brands` integration proxies brand images through
  `/api/brands/integration/<domain>/<image>` and reads this directory first,
  falling back to the `brands.home-assistant.io` CDN only when the file is
  absent. The directory existing is the entire opt-in —
  `Integration.has_branding` is `"brand" in self._top_level_files`, and no
  `manifest.json` key is involved.
  - **There is nothing to submit anywhere.** `home-assistant/brands` keeps a
    legacy `custom_integrations/` folder but **no longer accepts pull
    requests for custom integrations**. Never add a tracked TODO or a
    roadmap item for submitting artwork upstream; shipping `brand/` is the
    whole job. (See the
    [announcement](https://developers.home-assistant.io/blog/2026/02/24/brands-proxy-api/).)
  - Only these filenames are served (`ALLOWED_IMAGES` in
    `homeassistant/components/brands/const.py`), and a missing one falls back
    down a chain ending at `icon.png`: `icon.png`, `logo.png`, `icon@2x.png`,
    `logo@2x.png`, `dark_icon.png`, `dark_logo.png`, `dark_icon@2x.png`,
    `dark_logo@2x.png`. An `icon.svg` may be kept as the render source; HA
    ignores it.
  - HACS's own downloads panel may still show "icon not available" for
    integrations shipping local brand images — that is a HACS gap, not a
    packaging mistake.
  - **icon** — always **square** (256×256 / 512×512). Symbol only.
  - **logo** — always **rectangular / landscape** (e.g. 256×128 / 512×256).
    Wordmark or full brand mark.
  - Always ask the user for the source artwork; never generate placeholders.
    If asking is impossible, follow the autonomous fallback in
    [new-integration.md](./new-integration.md) — tracked TODO, prominent flag,
    release blocker.
- Release-please tags releases on merge to main.
