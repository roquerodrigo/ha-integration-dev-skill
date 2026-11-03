# Official contributor docs

The conventions in this skill sit on top of Home Assistant APIs that change
from release to release. When a task touches an API surface that the repo's
own guide and this skill do not settle — a platform the integration has never
used, a flow step, a new helper, a card API — read the current official page
instead of relying on remembered signatures.

## Fetching pages

- Base URL: `https://contributors.home-assistant.io/`. Every path below is
  relative to it.
- Each page has a Markdown version at `<page path>/index.md` — fetch that, not
  the rendered HTML. For a human-readable link, drop the `index.md`.
- Each area has an index of its pages: `core/llms.txt` (integrations and
  entities), `frontend/llms.txt` (cards and frontend), `marketplace/llms.txt`
  (HACS packaging and releases), and `llms.txt` for the whole site. Use them to
  find a page that is not listed here, or when a path below returns 404.
- Never load the `llms-full.txt` files; they hold the full text of an entire
  area. Fetch only the pages the change needs.
- Core docs describe `home-assistant/core`. Where a page asks for something
  that only exists in core (hassfest, the brands repository, `strings.json`
  instead of `translations/en.json`), apply the intent through this skill's
  custom-integration conventions.
- Check the repo's minimum Home Assistant version (`hacs.json`
  `homeassistant`) before adopting an API the page describes as new.
  `core/development/catching-up/index.md` lists the recent changes that affect
  custom integrations.

## Integrations

| Topic | Path |
| --- | --- |
| Setup, unload, `runtime_data` | `core/integrations/config-entries/index.md` |
| Setup failures and action exceptions | `core/integrations/setup-failures/index.md` |
| Config flow, flow steps | `core/integrations/config-flow/index.md`, `core/integrations/flow-steps/index.md` |
| Reauth and reconfigure | `core/integrations/reauth-reconfigure/index.md` |
| Options flow, subentries | `core/integrations/options-flow/index.md`, `core/integrations/subentries/index.md` |
| Input validation schemas | `core/development/input-validation/index.md` |
| Coordinator and polling | `core/integrations/fetching-data/index.md` |
| Manifest | `core/integrations/manifest/index.md` |
| Actions (`services.yaml`) | `core/integrations/actions/index.md` |
| Events | `core/integrations/events/index.md` |
| Triggers and conditions | `core/integrations/triggers-and-conditions/index.md` |
| Device and entity registry | `core/integrations/device-registry/index.md`, `core/integrations/entity-registry/index.md` |
| Discovery | `core/integrations/discovery/index.md`, `core/integrations/network-discovery/index.md` |
| Bluetooth | `core/integrations/bluetooth/index.md` |
| OAuth2 | `core/integrations/oauth2/index.md` |
| Diagnostics, repairs, system health | `core/integrations/diagnostics/index.md`, `core/integrations/repairs/index.md`, `core/integrations/system-health/index.md` |
| Translations | `core/integrations/translations/index.md` |
| API library expectations | `core/integrations/api-library/index.md` |
| Asyncio, blocking calls, thread safety | `core/internals/asyncio/index.md`, `core/internals/blocking-operations/index.md`, `core/internals/thread-safety/index.md` |
| Deprecations | `core/development/deprecating/index.md` |
| Quality scale | `core/quality-scale/rules/index.md` (see [quality-scale.md](./quality-scale.md)) |
| Tests | `core/development/tests/index.md` |

## Entities

Read `core/entities/entity-class/index.md` plus the page of the platform
being edited. Platform pages live at `core/entities/<domain>/index.md`, with
the underscores of the domain written as dashes: `binary_sensor` →
`core/entities/binary-sensor/index.md`, `alarm_control_panel` →
`core/entities/alarm-control-panel/index.md`.

| Topic | Path |
| --- | --- |
| Names and translation keys | `core/entities/naming/index.md` |
| Polling, push, availability | `core/entities/state-updates/index.md` |
| Adding, removing, subscriptions | `core/entities/lifecycle/index.md` |
| State and extra attributes | `core/entities/state-attributes/index.md` |
| Icons | `core/entities/icons/index.md` |
| Unique IDs, devices, registry defaults | `core/entities/registry/index.md` |

## Cards and frontend

| Topic | Path |
| --- | --- |
| Card API and sizing | `frontend/custom-ui/card/index.md` |
| Editor, stub config, picker, suggestions | `frontend/custom-ui/card-editor/index.md` |
| The `hass` object and its formatters | `frontend/reference/hass-object/index.md` |
| Context subscriptions | `frontend/architecture/hass-and-contexts/index.md` |
| Frontend translations | `frontend/architecture/translations/index.md` |
| Theme variables | `frontend/architecture/theming/index.md`, `marketplace/reference/theme-variables/index.md` |
| Dashboard resources and cache busting | `frontend/custom-ui/resources/index.md` |
| Card features, badges, strategies, views | `frontend/custom-ui/card-feature/index.md`, `frontend/custom-ui/badge/index.md`, `frontend/custom-ui/strategy/index.md`, `frontend/custom-ui/view/index.md` |

## HACS packaging

| Topic | Path |
| --- | --- |
| Integration packaging | `marketplace/guides/integrations/index.md` |
| Card packaging | `marketplace/guides/dashboard/index.md` |
| Repository structure | `marketplace/reference/repository-structure/index.md` |
| `hacs.json` | `marketplace/reference/hacs-json/index.md` |
| Validation | `marketplace/reference/validation/index.md` |
| Releases | `marketplace/guides/releases/index.md` |
