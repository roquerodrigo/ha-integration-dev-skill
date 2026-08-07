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
├── test_repairs.py          # only if repairs.py exists (issue registry)
└── test_translations.py     # locale key parity + no-empty-values
```

## Coverage gate

Enforce a minimum coverage via `pyproject.toml` (`--cov-fail-under=N`).

- **Default target: 90%** for integrations and companion SDKs alike, including
  hardware-heavy paths (BLE, USB, proprietary binary protocols) — mock the
  transport/byte layer rather than lowering the bar.
- **The repo's own configured gate is the source of truth.** Whatever
  `--cov-fail-under` the repo's `pyproject.toml` / `CODE_STYLE.md` sets wins —
  a higher gate (e.g. 95) *or* a lower one (e.g. an SDK at 60). The blueprint
  itself sets 90.
  In review, treat a gate below 90 as drift worth flagging (debt to repay), not
  a failure to block on; never "fail" a repo for honouring its own lower gate.

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

## Fixtures never contain real secrets

Fixtures, captures, and sample payloads use **synthetic values only** — never
real device credentials, cryptographic keys, unlock codes, tokens, MAC
addresses, or account/device nicknames. These repos are public and published
(GitHub, PyPI): a real key in a fixture is a credential leak, and the only
remediation is rotating the credential on the physical device — long after
every mirror has a copy.

- Build fixtures with realistic **shapes** and obviously fake values.
- When a real protocol capture seeds a fixture (golden tests), re-key and
  redact it before committing, and verify the parser still round-trips the
  sanitised bytes.
- If a real credential was ever committed, rotating it is part of the fix —
  rewriting git history alone does not un-leak it.

## Translation tests

`test_translations.py` parametrises over every locale file in `translations/`
and asserts its flattened key set is identical to `en.json`:

```python
@pytest.mark.parametrize("locale", OTHER_LOCALES)  # discovered from translations/
def test_translation_keys_match(locale):
    # load en.json and the locale file
    # assert flatten_keys(en) == flatten_keys(locale)
```

Also assert there are **no empty translation values** across every locale — an
empty string passes key-parity but renders as a blank label in the UI:

```python
def test_no_empty_translation_values():
    for path in translation_files():
        data = load(path)
        for key in flatten_keys(data):
            assert resolve(data, key), f"{path.name}: empty value for {key}"
```

## CI/CD pipeline

CI is composed from the **reusable workflows in `roquerodrigo/workflows`** —
do not hand-roll the jobs in each repo. A standard integration `ci.yml` calls:

- **lint** — `roquerodrigo/workflows/.github/workflows/python-lint.yml@main`:
  `uv run ruff check`, `uv run ruff format --check`, and `uv run mypy` (the
  mypy target comes from `[tool.mypy] files` in the consumer's
  `pyproject.toml`, so CI and local runs check the same paths).
- **tests** — `python-test.yml@main`: `uv run pytest`; the coverage gate lives
  in the consumer's `pyproject.toml`.
- **validate** — `home-assistant-validate.yml@main`: hassfest + HACS
  validation + a manifest×pyproject version check. Inputs cover the three
  consumer shapes: `hacs: false` for a **private** repo (HACS validation reads
  the repository through the public GitHub API), `hacs-category: plugin` for a
  Lovelace card, and `hassfest` / `version-check` opt-outs for repos with no
  manifest.
- **update-pr-branch** — `update-pr-branch.yml@main` keeps PRs in sync with
  `main`.

`lint` gates `tests` and `validate` via `needs:` so a single style issue
doesn't waste downstream minutes, and a `concurrency` block cancels superseded
pull-request runs.

Separate workflows in the same repo:

- **codeql** — `codeql.yml@main`, weekly security scan.
- **auto-assign** — `auto-assign.yml@main`, assigns issues/PRs to the owner.
- **release** (`release.yml`) — gated on a **successful CI run** rather than on
  the push itself (`workflow_run` on CI completion, `push` event only, so a
  green PR run cannot cut a release), calling `release-please.yml@main` with
  `release-token: ${{ secrets.RELEASE_PLEASE_PAT }}`. The reusable falls back
  to `GITHUB_TOKEN` when the secret is unset — but pushes made with that token
  trigger no workflows, leaving the release PR with stale checks.

## Running tests locally

```bash
# first time setup
uv sync --group dev --group lint

# verify — scripts/lint is a thin wrapper that only chains these commands;
# use it when the repo ships one, or run them directly:
uv run ruff format --check .
uv run ruff check .
uv run mypy custom_components/<domain>   # or src/ for an SDK
uv run pytest
```
