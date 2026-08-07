# Code review checklist

Use this when reviewing a PR against a HA integration that follows the
conventions in this skill. Items are grouped by area; cite the file and line
when flagging a violation. Hard rules first, judgment-calls later.

## Coordinator (`coordinator.py`, `<purpose>_coordinator.py`)

- Return type is `DataUpdateCoordinator[PayloadType]` with a concrete
  TypedDict or dataclass — never `Any` or bare `dict`.
- `_async_update_data()` maps auth errors → `ConfigEntryAuthFailed` and
  comms errors → `UpdateFailed`. No raw upstream/SDK exception escapes. A
  documented grace-period / last-known-data fallback (returning the previous
  payload for a known-flaky upstream) is an accepted alternative to raising
  `UpdateFailed` — don't flag it when the degradation strategy is explicit.
- First setup uses `await coordinator.async_config_entry_first_refresh()`,
  not `async_refresh()`.
- `always_update=False` whenever the payload supports `__eq__` cleanly.
- A PR adding a second coordinator must justify it by **different polling
  cadence** (not per-entity or per-platform splitting), put it in its own
  `<purpose>_coordinator.py`, and surface it as a distinct field in
  `runtime_data`.

## API client (`api.py`, `api/`)

- SDK exceptions are mapped to the `<Domain>ApiClientError` hierarchy; no raw
  SDK exception escapes the wrapper.
- `async_connect` (or whatever the config flow calls to validate) issues a
  real round-trip, not just a lazy constructor. Flag a `try/except` wrapping
  only `SomeClient(url=...)` — lazy clients never raise there, so it's dead
  code and the flow reports invalid input as valid.
- SDK data is read through the public API (`obj["field"]`, a property, a
  method), not private attributes. Flag `obj._private...` with a `# noqa:
  SLF001` where a documented accessor exists.
- No dead fields in the payload: every value computed into the coordinator
  payload / data TypedDict is consumed by an entity, attribute, or diagnostic.
  Flag a field that is computed and typed but never surfaced.

## Config flow (`config_flow.py`)

- Steps wired where applicable: `async_step_user`, `async_step_reauth` →
  `async_step_reauth_confirm`, `async_step_reconfigure`, and
  `async_get_options_flow`. **Reauth is not mandatory** — an unauthenticated
  source (local socket, anonymous HTTP) legitimately omits the reauth steps and
  the `AuthenticationError` type. Don't flag their absence; do flag reauth
  scaffolding left in for a source that can't authenticate.
- `unique_id` set from a stable identifier; aborts on duplicate.
- Entry **title** is a stable, human-distinguishing value (the host, socket
  path, account, or device name) — not a hard-coded constant. Two entries of
  the same integration must be tellable apart in the UI. Flag a fixed string
  literal as the title when the integration supports multiple entries.
- `_validate` and the schema-builder helper are extracted, not inlined per
  step. The builder is often credential-shaped, but may instead build a
  host/socket field — match the integration's actual input.
- Every error/abort string resolves through `translations/`, not hardcoded.

## File organisation

- One class per file — no exceptions. TypedDicts and dataclasses each get
  their own file under `data/` (a package). `type` aliases live in
  `data/__init__.py`. Flag any module with two or more top-level classes —
  including a flat `data.py` stacking several TypedDicts/dataclasses, **even
  when the repo's own docs still show the flat form**: this is a blueprint
  invariant, not a style preference (report it as migration debt, not a
  blocker, when the PR doesn't touch `data.py`).
- A platform with multiple entities is a package
  (`sensor/__init__.py` + `sensor/<entity>.py`), not a single
  `sensor.py` stacking the classes.
- `<platform>/__init__.py` contains only `async_setup_entry`.
- `api.py` may be a flat file or an `api/` package (`client.py` +
  `errors.py` + ...) — accept whichever the repo already uses; don't ask
  for a split that doesn't exist yet.
- Exceptions live either under `exceptions/` (one per file) **or**
  per-subsystem in `<subsystem>/errors.py`. Flag mixing both styles in the
  same integration.

## Entities

- One class per entity. No generic class parametrised by an
  `EntityDescription` subclass with `value_fn` / `action_fn` callables.
- Naming `<Domain><Name><Platform>` (e.g. `<Domain>BatterySensor`).
- State exposed via `@property`, not `_attr_*` assigned in `__init__`.
  `__init__` omitted when it would only call `super().__init__(...)`.
- `unique_id` exposed as a `@property` (the only permitted form). Flag any
  `_attr_unique_id` assigned in `__init__`.
- Base entity sets `_attr_attribution`, `_attr_has_entity_name = True`, and
  `device_info` as a `@property`.
- Maintenance/telemetry entities (signal strength, firmware version,
  secondary battery) carry `_attr_entity_category = EntityCategory.DIAGNOSTIC`.
  User-tunable settings exposed as entities use `EntityCategory.CONFIG`.
- Rarely-useful entities set `_attr_entity_registry_enabled_default = False`.
- No explicit `_attr_should_poll = False` — already inherited from
  `CoordinatorEntity`.
- Accessors that read `coordinator.data` narrow it through a local
  `<Domain>Payload | None` and guard `is None` (it is `None` before the first
  refresh, despite the non-`Optional` generic). Flag direct
  `self.coordinator.data["..."]` indexing with no guard, and flag a guard
  written without the `| None` annotation (strict mypy marks it `unreachable`).
- Member order inside the class: `_attr_*` constants → `__init__` (if any)
  → `@property`s → HA lifecycle coroutines (`async_added_to_hass`, ...)
  → other public methods → private methods.

## Async patterns

- No blocking I/O inside a coroutine — sync SDK calls wrapped in
  `await hass.async_add_executor_job(fn, *args)`.
- `await asyncio.sleep(...)` only — never `time.sleep()` in a coroutine.
- `hass.async_create_task(coro)` over bare `asyncio.create_task(coro)`.
- `homeassistant.util.dt.utcnow()` over `datetime.now(UTC)` (HA helper is
  patchable in tests).

## Runtime data

- All state lives on `entry.runtime_data: <Domain>Data`. No
  `hass.data[DOMAIN]`.
- `<Domain>Data` is a `@dataclass` defined in `data.py`.

## Typing

- No `typing.Any`, `object` as a value, bare `dict`/`list`/`tuple`/`set`,
  `dict[str, Any]`, or `Mapping[str, Any]`.
- `TypedDict` for known JSON/dict shapes — canonical home is `data.py`.
- `cast("TypedDictName", value)` at HA framework boundaries that return
  permissive types (`entry.data`, etc.).
- Any `# type: ignore[override]` carries a one-line reason.
- On `requires-python >= 3.14`, `except A, B:` without parentheses (PEP 758) is
  valid — do not flag it as a bug.

## Imports & module hygiene

- `from __future__ import annotations` at the top of every `.py`.
- Type-only imports live inside `if TYPE_CHECKING:`.
- Same-package imports are relative (`from .module import ...`).
- `noqa` only for unavoidable framework constraints (e.g. `ARG001`).

## Constants

- New values reused across modules or representing a domain concept (default
  scan interval, MIN/MAX bounds, multi-file payload keys) live in `const.py`.
- One-shot literals stay inline — no premature extraction.

## Docstrings & comments

- Every new public class, function, method, and `@property` has a docstring.
- Single-line preferred. Multi-line docstrings put the summary on the
  **second** line (D213) and have no blank line before a class docstring
  (D211).
- New comments only explain *why* a non-obvious choice was made. Comments
  describing *what* the code does, or section dividers like
  `# --- section ---`, should be flagged.

## Logging

- Uses package-level `LOGGER` from `const.py`.
- Lazy `%`-formatting, never f-strings. No secrets in any log line.
- Level matches the event: `debug` (poll summaries), `info` (lifecycle),
  `warning` (recoverable), `error`/`exception` (unrecoverable).
- Inside `except:` blocks, prefer `LOGGER.exception("msg")` over
  `LOGGER.error("msg", exc_info=True)`.

## Translations

- Every new user-facing string has an entry in `en.json` AND every other
  locale file under `translations/`.
- `test_translations.py` still passes (locale key parity).

## Diagnostics & repairs

- Any new sensitive field added to the payload appears in `TO_REDACT` in
  `diagnostics.py`. An **empty `TO_REDACT` is valid** when the entry holds no
  secrets (e.g. just a socket path) — don't demand redaction of non-secrets.
- `repairs.py` and `test_repairs.py` exist only if the integration actually
  registers issues. If it doesn't, both should have been deleted (blueprint
  sample code) — flag a `repairs.py` that registers nothing.
- New issues registered via the repairs flow have translated strings under
  `issues.<issue_id>`.

## Exception hierarchy

- New exceptions subclass `<Domain>ApiClientError` (or one of its
  `Authentication` / `Communication` subclasses), live one-per-file under
  `exceptions/`, and are re-exported from `exceptions/__init__.py`.

## Tests

- New code is covered — the coverage gate in `pyproject.toml` still passes.
- No real network calls. The API client is mocked at class level at both
  the `__init__` and `config_flow` import sites.
- Async methods mocked with `AsyncMock`.

## Quality scale

- `quality_scale.yaml` exists (the blueprint ships it; a fork that deleted it
  regressed — flag its absence).
- Each rule's `status` reflects reality: no rule claims `done` for behaviour
  the code doesn't implement. `exempt` carries a `comment` justifying why it
  doesn't apply (unauthenticated source → `reauthentication-flow` exempt; SDK
  owns its connector → `inject-websession` exempt). A half-done rule is `todo`,
  not `done`.
- Objective spot-checks for the rules most often claimed falsely:
  - `parallel-updates: done` requires a module-level `PARALLEL_UPDATES`
    constant in **every** platform module — no constant, no `done`.
  - `exception-translations: done` requires raising `HomeAssistantError` (or
    subclasses) with `translation_domain`/`translation_key`, plus the matching
    `exceptions` section in the translation files.
  - `repair-issues: done` requires the issue-raising helper to actually be
    called somewhere — a scaffold function with no call site is `todo`.

## Manifest & release metadata

- `manifest.json` `version` bumped per SemVer when behaviour changes
  (release-please usually handles this from the commit message).
- `codeowners` still accurate.
- `hacs.json` `homeassistant` minimum reflects any new HA API usage.

## Commit message

- Conventional Commits format: `feat`/`fix`/`perf`/`refactor`/...
- Subject line is imperative, lowercase, no trailing period.
- `BREAKING CHANGE:` footer present for any major-bump change.

## Verification gate

- The lint commands and `pytest` were run locally and passed before review —
  invoked directly (`ruff format --check`, `ruff check`, `mypy <path>`,
  `pytest`), prefixed with `uv run` in uv-managed repos. No `scripts/lint`
  wrapper required.
