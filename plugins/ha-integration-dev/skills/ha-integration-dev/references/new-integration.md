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

## Step 2: Clean blueprint artifacts

These files carry state from the blueprint's release history and must be
reset — the global rename in Step 3 will not fix them:

- **`.release-please-manifest.json`** — set version to `"0.0.0"`.
- **`CHANGELOG.md`** — empty the file (keep only the `# Changelog` heading).
  The first release-please PR will populate it.
- **`README.md`** — rewrite from scratch for the new integration. The
  blueprint's README describes setup steps, entity tables, and architecture
  specific to the blueprint; none of that applies. At minimum include: what
  the integration does, how to install, and how to configure.
- **`brand/`** — **ask the user** for the icon and logo images (or a URL to
  download them from). Do not generate placeholder artwork or reuse the
  blueprint's assets. The two assets have different aspect ratios:
  - **`icon.png`** + **`icon@2x.png`** — must be **square** (256×256 and
    512×512). This is the small symbol shown in device lists and entity
    cards. Crop or pad the source image to a 1:1 ratio.
  - **`logo.png`** + **`logo@2x.png`** — must be **rectangular / landscape**
    (e.g. 256×128 and 512×256). This is the wider mark shown in the
    integrations page and HACS. Keep the original aspect ratio of the brand
    wordmark; do not force it into a square.
  - **`icon.svg`** — vector version of the icon (square).
  If the user provides a single image, ask whether it should be used as icon,
  logo, or both (and which crop to apply).
- **`.idea/`** — either delete the directory entirely (it can be regenerated
  by the IDE) or rename `ha-integration-blueprint.iml` →
  `ha-<new-name>.iml` and update `modules.xml` to match.
- **`uv.lock`** — delete it; `uv sync` will regenerate it with the correct
  dependencies after you edit `pyproject.toml`.

## Step 3: Rename the domain

The blueprint uses `integration_blueprint` as domain and `IntegrationBlueprint`
as the class prefix. Replace globally:

1. Rename `custom_components/integration_blueprint/` →
   `custom_components/<new_domain>/`
2. Find-and-replace in all `.py`, `.json`, `.yaml`, `.md` files:
   - `integration_blueprint` → `<new_domain>`
   - `IntegrationBlueprint` → `<NewDomain>`
   - `Integration Blueprint` → `<New Name>`
   - `ha-integration-blueprint` → `ha-<new-name>`

## Step 4: Update manifest.json

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

## Step 5: Update hacs.json

```json
{
  "name": "<New Name>",
  "homeassistant": "<min-ha-version>",
  "hacs": "<min-hacs-version>"
}
```

## Step 6: Update pyproject.toml

- Change `name`, `version` (to `"0.1.0"`), `description`.
- Add the SDK to dev dependencies.
- Update `--cov=custom_components/<new_domain>` in pytest addopts.
- Update `files = ["custom_components/<new_domain>"]` in the mypy section.

## Step 7: Implement the API client

Replace the sample `api.py` with a wrapper around the real SDK. The wrapper
must:

1. Map SDK exceptions to the custom exception hierarchy.
2. Use `hass.async_add_executor_job()` for sync SDKs.
3. Never expose raw SDK exceptions above the API boundary.

## Step 8: Define data types

In `data.py`:
- `<NewDomain>ConfigData(TypedDict)` — credentials from config flow
- `<NewDomain>OptionsData(TypedDict, total=False)` — from options flow
- `<NewDomain>Data(@dataclass)` — runtime data (client, coordinator, integration)
- `<NewDomain>ConfigEntry = ConfigEntry[<NewDomain>Data]`
- Payload TypedDict(s) for coordinator return type

## Step 9: Implement the coordinator

In `coordinator.py`:
- Set the return type: `DataUpdateCoordinator[PayloadType]`
- Implement `_async_update_data()` → call API client, return typed payload
- Map auth errors → `ConfigEntryAuthFailed`
- Map comms errors → `UpdateFailed`

## Step 10: Add entity platforms

For each platform:
1. Create entity class(es) — one class per entity.
2. Implement `async_setup_entry()` that instantiates and registers entities.
3. Add the platform to `PLATFORMS` in `__init__.py`.
4. Add translations for entity names and enum states.

## Step 11: Add translations

Update every supported locale file under `translations/` with:
- Config flow step titles, descriptions, data labels
- Error and abort messages
- Entity names and state labels

At minimum ship `en.json`; add other locales as needed. Whatever set you ship,
`test_translations.py` should enforce key parity across all of them.

## Step 12: Write tests (same pass as Steps 7–11)

Tests are not a separate step — write them alongside each module as you
implement it. By the time you reach this checklist item, every `.py` file
under `custom_components/<new_domain>/` should already have its test
counterpart. This step is a verification that nothing was missed.

Mirror the production layout in `tests/`. Target 90% coverage. At minimum:
- `test_init.py` — setup, unload, reload
- `test_config_flow.py` — all flow steps and error paths
- `test_coordinator.py` — data fetch and error mapping
- `test_api.py` — exception translation
- One `test_<platform>.py` per platform
- `test_translations.py` — locale key parity
- `test_diagnostics.py` — redaction

## Step 13: Verify

Run the tools directly (no `scripts/lint` wrapper):

```bash
uv run ruff format --check .
uv run ruff check .
uv run mypy custom_components/<new_domain>
uv run pytest
```

## Step 14: Set up GitHub

1. Create the repo at `<your-github-org>/ha-<new-name>` (**public**).
2. Add GitHub topics — at minimum `home-assistant` and `hacs` (HACS
   validation will fail without valid topics).
3. Add the `RELEASE_PLEASE_PAT` repo secret. The PAT is available as
   the `RELEASE_PLEASE_PAT` environment variable — set it directly:
   `gh secret set RELEASE_PLEASE_PAT --repo <org>/ha-<new-name> --body "$RELEASE_PLEASE_PAT"`
   Without it, the `release` job fails with
   `Input required and not supplied: token`.
4. Push the initial commit.
5. Add branch protection on `main` requiring CI green.
6. `.github/workflows/ci.yml` is already in the blueprint.
7. Enable Dependabot.
8. Create the first release via release-please.
