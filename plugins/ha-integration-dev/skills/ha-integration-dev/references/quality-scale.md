# Integration quality scale

Home Assistant grades core integrations on a quality scale of 54 rules in four
cumulative tiers: Bronze, Silver, Gold, Platinum. A custom integration has no
official tier, but the rules are a good lens for review: they name the user
experience gaps that this skill's conventions do not always cover.

Use the scale as a **review lens, not a target**. Report which rules the code
misses and say which tier each belongs to; do not grow the scope of a change to
reach a tier the user did not ask for. When the user does ask for a tier, every
rule of that tier and of the tiers below it is part of the work.

## Source of truth

The tables below follow `core/quality-scale/rules/index.md` (see
[upstream-docs.md](./upstream-docs.md) for fetching). Rules are added and
reworded over time: before a full quality review, fetch that page and trust it
over this file. Each rule has its own page at `core/quality-scale/<rule>/index.md`
with the reasoning, an example and the accepted exemptions — read it before
flagging a borderline case.

## Applying core rules to a custom integration

Some rules assume core infrastructure. Apply their intent instead:

| Rule | In a custom integration |
| --- | --- |
| `brands` | Brand images ship in `custom_components/<domain>/brand/` (see "HACS requirements" in [architecture.md](./architecture.md)); there is nothing to submit to `home-assistant/brands`. |
| `docs-*` | The documentation is the repo README, not a page on home-assistant.io. |
| `integration-owner` | `codeowners` in `manifest.json` names the maintainer. |
| `dependency-transparency` | The SDK is on PyPI and built and published from public CI (see [sdk.md](./sdk.md)). |
| `test-coverage` | Core asks for more than 95% per module; the repo's own `--cov-fail-under` still decides the gate (see [testing.md](./testing.md)). Report a lower gate as a Silver gap, not a failure. |
| `entity-translations`, `exception-translations`, `icon-translations` | Strings live in `translations/en.json`; core's `strings.json` does not exist here. Icons go in `icons.json`. |

## Bronze

| Rule | What it asks |
| --- | --- |
| `action-setup` | Actions are registered in `async_setup`, not per config entry. |
| `appropriate-polling` | A polling integration uses an interval that fits its data. |
| `brands` | The integration has a logo and icon. |
| `common-modules` | The coordinator lives in `coordinator.py` and the base entity in `entity.py`. |
| `config-flow` | Set up from the UI; fields have a `data_description`; `ConfigEntry.data` and `ConfigEntry.options` are used for what each is for. |
| `config-flow-test-coverage` | The config flow has full test coverage, including recovery from errors. |
| `dependency-transparency` | Dependencies are open source, on PyPI, and built and published in public CI. |
| `docs-actions` | The documentation describes the actions. |
| `docs-conditions` | The documentation describes the conditions. |
| `docs-high-level-description` | The documentation describes what the integration connects to. |
| `docs-installation-instructions` | The documentation has step-by-step setup instructions, with prerequisites. |
| `docs-removal-instructions` | The documentation explains how to remove the integration. |
| `docs-triggers` | The documentation describes the triggers. |
| `entity-event-setup` | Entities subscribe in `async_added_to_hass` and unsubscribe when removed. |
| `entity-unique-id` | Every entity has a unique ID. |
| `has-entity-name` | Entities set `has_entity_name = True` and are named in context of their device. |
| `runtime-data` | Runtime data is stored in `ConfigEntry.runtime_data`. |
| `test-before-configure` | The config flow tests the connection before it creates the entry. |
| `test-before-setup` | Setup checks that it works and raises the right exception when it does not. |
| `unique-config-entry` | The same device or service cannot be set up twice. |

## Silver

| Rule | What it asks |
| --- | --- |
| `action-exceptions` | Actions raise `ServiceValidationError` or `HomeAssistantError` when they fail. |
| `config-entry-unloading` | The entry can be unloaded and reloaded without a restart. |
| `docs-configuration-parameters` | The documentation describes every option of the options flow. |
| `docs-installation-parameters` | The documentation describes every field asked for during setup. |
| `entity-unavailable` | Entities become unavailable when the device or service cannot be reached. |
| `integration-owner` | The integration has at least one owner. |
| `log-when-unavailable` | A lost connection is logged once, and its recovery once. |
| `parallel-updates` | Every entity platform sets `PARALLEL_UPDATES`. |
| `reauthentication-flow` | Users can enter new credentials without removing the integration. |
| `test-coverage` | The tests cover more than 95% of every module. |

## Gold

| Rule | What it asks |
| --- | --- |
| `devices` | Entities are grouped in devices, with complete device information. |
| `diagnostics` | Users can download diagnostics, with sensitive data redacted. |
| `discovery` | Devices are discovered, so users do not have to look them up. |
| `discovery-update-info` | Discovery updates the address of a known device when it changes. |
| `docs-data-update` | The documentation explains how and how often data is updated. |
| `docs-examples` | The documentation gives automation examples or blueprints. |
| `docs-known-limitations` | The documentation lists the known limitations. |
| `docs-supported-devices` | The documentation lists known supported and unsupported devices. |
| `docs-supported-functions` | The documentation describes the entities and functionality provided. |
| `docs-troubleshooting` | The documentation helps users solve common problems themselves. |
| `docs-use-cases` | The documentation shows what users can do with the integration. |
| `dynamic-devices` | Devices that appear after setup are added without a reload. |
| `entity-category` | Configuration and diagnostic entities have an entity category. |
| `entity-device-class` | Entities use a device class wherever one fits. |
| `entity-disabled-by-default` | Entities that few users need are disabled by default. |
| `entity-translations` | Entity names are translated. |
| `exception-translations` | Errors shown to users are translated. |
| `icon-translations` | Entity icons are defined in `icons.json`. |
| `reconfiguration-flow` | Users can change the setup of an entry without removing it. |
| `repair-issues` | Repair issues are raised for problems users can fix. |
| `stale-devices` | Devices that are gone are removed, or can be removed by the user. |

## Platinum

| Rule | What it asks |
| --- | --- |
| `async-dependency` | The library uses asyncio. |
| `inject-websession` | The library accepts Home Assistant's web session (`async_get_clientsession`). |
| `strict-typing` | The integration is fully typed and checked with strict mypy. |

## Exemptions

A rule that cannot apply is exempt, not missed: `discovery` for a cloud-only
service, `reauthentication-flow` for a source without credentials, `devices`
for an integration that represents no device, the `docs-*` rules for a feature
the integration does not have. Name the exemption and its reason in the report
instead of counting the rule as a gap.
