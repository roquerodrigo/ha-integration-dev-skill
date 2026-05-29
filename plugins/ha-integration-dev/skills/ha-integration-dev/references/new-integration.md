# Creating a new integration from the blueprint

This guide assumes you start from a blueprint repository (e.g.
`ha-integration-blueprint`) — a pre-wired skeleton that already follows the
patterns in [architecture.md](./architecture.md) and
[coding-conventions.md](./coding-conventions.md).

Throughout this guide, `<new-name>` is the kebab-case integration name
(`my-device`), `<new_domain>` is the snake_case Home Assistant domain
(`my_device`), `<NewDomain>` is the PascalCase class prefix (`MyDevice`), and
`<New Name>` is the human-readable label (`My Device`).

## Step 1: Fork the blueprint

```bash
cp -r ha-integration-blueprint ha-<new-name>
cd ha-<new-name>
rm -rf .git
git init
```

## Step 2: Rename the domain

The blueprint uses `integration_blueprint` as domain and `IntegrationBlueprint`
as the class prefix. Replace globally:

1. Rename `custom_components/integration_blueprint/` →
   `custom_components/<new_domain>/`
2. Find-and-replace in all `.py`, `.json`, `.yaml`, `.md` files:
   - `integration_blueprint` → `<new_domain>`
   - `IntegrationBlueprint` → `<NewDomain>`
   - `Integration Blueprint` → `<New Name>`
   - `ha-integration-blueprint` → `ha-<new-name>`

## Step 3: Update manifest.json

```json
{
  "domain": "<new_domain>",
  "name": "<New Name>",
  "codeowners": ["@<your-github-handle>"],
  "config_flow": true,
  "documentation": "https://github.com/<your-github-org>/ha-<new-name>",
  "iot_class": "<local_polling|cloud_polling|cloud_push|local_push>",
  "issue_tracker": "https://github.com/<your-github-org>/ha-<new-name>/issues",
  "requirements": ["<sdk-package>==<version>"],
  "version": "0.1.0"
}
```

## Step 4: Update hacs.json

```json
{
  "name": "<New Name>",
  "homeassistant": "<min-ha-version>",
  "hacs": "<min-hacs-version>"
}
```

## Step 5: Update pyproject.toml

- Change `name`, `version`, `description`.
- Add the SDK to dev dependencies.
- Update `--cov=custom_components/<new_domain>` in pytest addopts.
- Update `files = ["custom_components/<new_domain>"]` in the mypy section.

## Step 6: Implement the API client

Replace the sample `api.py` with a wrapper around the real SDK. The wrapper
must:

1. Map SDK exceptions to the custom exception hierarchy.
2. Use `hass.async_add_executor_job()` for sync SDKs.
3. Never expose raw SDK exceptions above the API boundary.

## Step 7: Define data types

In `data.py`:
- `<NewDomain>ConfigData(TypedDict)` — credentials from config flow
- `<NewDomain>OptionsData(TypedDict, total=False)` — from options flow
- `<NewDomain>Data(@dataclass)` — runtime data (client, coordinator, integration)
- `<NewDomain>ConfigEntry = ConfigEntry[<NewDomain>Data]`
- Payload TypedDict(s) for coordinator return type

## Step 8: Implement the coordinator

In `coordinator.py`:
- Set the return type: `DataUpdateCoordinator[PayloadType]`
- Implement `_async_update_data()` → call API client, return typed payload
- Map auth errors → `ConfigEntryAuthFailed`
- Map comms errors → `UpdateFailed`

## Step 9: Add entity platforms

For each platform:
1. Create entity class(es) — one class per entity.
2. Implement `async_setup_entry()` that instantiates and registers entities.
3. Add the platform to `PLATFORMS` in `__init__.py`.
4. Add translations for entity names and enum states.

## Step 10: Add translations

Update every supported locale file under `translations/` with:
- Config flow step titles, descriptions, data labels
- Error and abort messages
- Entity names and state labels

At minimum ship `en.json`; add other locales as needed. Whatever set you ship,
`test_translations.py` should enforce key parity across all of them.

## Step 11: Write tests

Mirror the production layout in `tests/`. Target 90% coverage. At minimum:
- `test_init.py` — setup, unload, reload
- `test_config_flow.py` — all flow steps and error paths
- `test_coordinator.py` — data fetch and error mapping
- `test_api.py` — exception translation
- One `test_<platform>.py` per platform
- `test_translations.py` — locale key parity
- `test_diagnostics.py` — redaction

## Step 12: Verify

```bash
scripts/lint && pytest
```

## Step 13: Set up GitHub

1. Create the repo at `<your-github-org>/ha-<new-name>`.
2. Push the initial commit.
3. Add branch protection on `main` requiring CI green.
4. Copy `.github/workflows/ci.yml` (already in the blueprint).
5. Enable Dependabot.
6. Create the first release via release-please.
