# Testing patterns

## Structure

Tests live in `tests/`, mirroring the production layout:

```
tests/
├── conftest.py              # shared fixtures
├── __init__.py
├── test_init.py             # setup/unload/reload entry
├── test_config_flow.py      # user/reauth/reconfigure flows
├── test_options_flow.py     # options flow
├── test_coordinator.py      # data update, error mapping
├── test_api.py              # API client, exception mapping
├── test_entity.py           # base entity class
├── test_sensor.py           # one file per platform
├── test_binary_sensor.py
├── test_switch.py
├── test_button.py
├── test_diagnostics.py      # redaction, payload shape
├── test_repairs.py          # issue registry
└── test_translations.py     # locale key parity
```

## Coverage gate

Enforce a minimum coverage via `pyproject.toml` (`--cov-fail-under=90`).

- **90% for every repo** — integrations and companion SDKs alike, including
  hardware-heavy paths (BLE, USB, proprietary binary protocols). Mock the
  transport/byte layer rather than lowering the gate; a sub-90 gate is debt to
  repay, not an acceptable resting state.

## conftest.py pattern

```python
from __future__ import annotations

import copy
from typing import TYPE_CHECKING
from unittest.mock import AsyncMock, patch

import pytest

if TYPE_CHECKING:
    from collections.abc import Generator

pytest_plugins = "pytest_homeassistant_custom_component"

SAMPLE_PAYLOAD = { ... }  # realistic sample data

@pytest.fixture
def sample_payload() -> dict:
    return copy.deepcopy(SAMPLE_PAYLOAD)

@pytest.fixture
def enable_custom_integrations(hass) -> None:
    from homeassistant.loader import DATA_CUSTOM_COMPONENTS
    hass.data.pop(DATA_CUSTOM_COMPONENTS, None)

@pytest.fixture
def mock_api_client(sample_payload) -> Generator:
    with patch(
        "custom_components.<domain>.<Domain>ApiClient"
    ) as mock_class:
        instance = mock_class.return_value
        instance.async_get_data = AsyncMock(return_value=sample_payload)
        yield instance

@pytest.fixture
async def setup_integration(hass, mock_api_client, enable_custom_integrations):
    from pytest_homeassistant_custom_component.common import MockConfigEntry
    from custom_components.<domain>.const import DOMAIN

    entry = MockConfigEntry(
        domain=DOMAIN,
        data={...},
        unique_id="...",
    )
    entry.add_to_hass(hass)
    await hass.config_entries.async_setup(entry.entry_id)
    await hass.async_block_till_done()
    return entry
```

## Mocking strategy

- Mock the API client at class level via `patch()`.
- Use `AsyncMock` for all async methods.
- For integrations with SDKs, patch both `__init__.<Domain>ApiClient` AND
  `config_flow.<Domain>ApiClient` (the import site).
- Use factory functions for SDK data objects (e.g. `make_client()`,
  `make_node()`) when the SDK exposes dataclasses.
- Never call real network APIs in tests.

## Translation tests

`test_translations.py` parametrises over every non-default locale file in
`translations/` and asserts its flattened key set is identical to `en.json`:

```python
@pytest.mark.parametrize("locale", OTHER_LOCALES)  # discovered from translations/
def test_translation_keys_match(locale):
    # load en.json and the locale file
    # assert flatten_keys(en) == flatten_keys(locale)
```

## CI/CD pipeline

Recommended jobs for a Home Assistant integration repo (extract into reusable
workflows under `<your-org>/.github` once the patterns stabilise):

- **lint**: `ruff format --check`, `ruff check`, `mypy`
- **tests**: `pytest` with the coverage gate from `pyproject.toml`
- **validate**: `home-assistant/actions/hassfest` + `hacs/action`
- **codeql**: weekly security scan
- **release**: `googleapis/release-please-action` — parses Conventional Commits
  to bump version, update `manifest.json`, and generate `CHANGELOG.md`
- **auto-assign**: assigns each PR to the codeowner
- **update-pr-branch**: keeps PRs in sync with `main`

`lint` should gate `tests`, `validate`, and `release` so a single style issue
doesn't waste downstream minutes.

## Running tests locally

```bash
# first time setup
uv sync --group dev --group lint

# verify — run the tools directly (no scripts/lint wrapper)
uv run ruff format --check .
uv run ruff check .
uv run mypy custom_components/<domain>   # or src/ for an SDK
uv run pytest
```
