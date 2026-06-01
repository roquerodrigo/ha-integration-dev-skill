# Coding conventions

## Language

- Code is written in English: file names, class names, variable names,
  dictionary keys, identifier strings.
- User-facing strings live in `translations/<locale>.json` only.

## Typing

Strict typing enforced by `mypy --strict`. No exceptions.

Banned: `typing.Any`, `object` as a value type, bare `dict` / `list` /
`tuple` / `set`, `dict[str, Any]`, `Mapping[str, Any]`.

Required:
- `TypedDict` for known dict / JSON shapes (one per file under `data/`).
- `@dataclass` for structured records (e.g. runtime data).
- Named `type` aliases for recursive / shared shapes: `JsonPrimitive`,
  `JsonValue`, `JsonObject` in `data/__init__.py`.
- `frozenset[str]` / `tuple[str, ...]` for fixed string collections.
- `cast("TypedDictName", value)` at HA framework boundaries that hand us a
  permissive type (e.g. `entry.data` → `MappingProxyType[str, Any]`).

When narrowing an HA callback signature, add `# type: ignore[override]` with
a one-line comment.

## Naming

- Public classes prefixed with `<Domain>` — a PascalCase form of the
  integration's domain (e.g. `IntegrationBlueprint` for domain
  `integration_blueprint`).
- Platform entities end with the platform type: `…Sensor`,
  `…BinarySensor`, `…Switch`, `…Button`, `…Lock`.
- Exception classes end with `Error`.
- Private attributes / functions prefixed with `_`.

## File organisation

See `architecture.md` for the full layout. The three load-bearing rules:

- **One class per file — no exceptions.** TypedDicts and dataclasses each
  get their own file under `data/` (a package with `__init__.py`
  re-exporting public symbols). `type` aliases live in `data/__init__.py`.
  Helper functions may live alongside the single class that uses them
  (e.g. `_verify_response` next to the API client in `api.py`).
- **One entity per class.** Never share a generic class parametrised by an
  `EntityDescription` subclass with callable fields like `value_fn` or
  `action_fn`. Encode behaviour directly via `@property` and class-level
  `_attr_*` constants.
- **Same-subject classes group into a package.** A platform with a single
  entity stays as a flat file (e.g. `switch.py`). A platform with multiple
  entities becomes a package: `sensor/__init__.py` containing only
  `async_setup_entry`, plus one file per entity
  (`sensor/temperature.py`, `sensor/battery.py`, ...). The same applies to
  any other group of related classes — split into a package, never stack
  them in one module.

## Constants

- A value lives in `const.py` if it is **reused across modules** or is a
  **domain concept** (e.g. `DEFAULT_SCAN_INTERVAL`, `MIN_TEMPERATURE`,
  payload keys read in more than one file).
- Literals used exactly once stay inline — no premature extraction.
- `DOMAIN`, `LOGGER`, and `ATTRIBUTION` always live in `const.py`.

## File / class member order

Within an entity class, declare members top-to-bottom in this order:

1. Class-level `_attr_*` constants (`_attr_has_entity_name`,
   `_attr_attribution`, etc.).
2. `__init__` (only when needed — see "Properties and `__init__`" below).
3. `@property` methods, public first then private.
4. HA lifecycle coroutines (`async_added_to_hass`, `async_will_remove_from_hass`).
5. Other public methods.
6. Private methods (`_method`).

## Properties and `__init__`

- Always prefer `@property` over assigning `_attr_*` values in `__init__`.
- When `__init__` would only call `super().__init__(...)`, omit it.
- Class-level constants like `_attr_attribution = ATTRIBUTION` are fine.
- **`unique_id` must be a `@property` — this is the only permitted form.**
  Expose it as a computed property returning a stable per-instance key:
  ```python
  @property
  def unique_id(self) -> str:
      """Return a stable unique id for this entity."""
      return f"{self.coordinator.config_entry.entry_id}_battery"
  ```
  Do **not** assign `_attr_unique_id` in `__init__`. If the only reason an
  `__init__` exists is to set `_attr_unique_id`, drop the `__init__` and use the
  property instead. (This matches the blueprint, which wires `unique_id` as a
  `@property` everywhere.) This is a **hard blueprint invariant** — it holds
  even if a repo's older `CODE_STYLE.md` still permits `_attr_unique_id`; migrate
  such code to the property form rather than treating the repo style as an
  override (unlike the coverage gate, which the repo *does* get to set).

## Entity categories and registry defaults

- Use `EntityCategory.DIAGNOSTIC` for maintenance/telemetry entities
  (signal strength, firmware version, secondary battery readings).
- Use `EntityCategory.CONFIG` for user-tunable settings exposed as entities.
- Set `_attr_entity_registry_enabled_default = False` for entities that are
  rarely useful (so they exist but stay hidden until enabled).
- Don't set `_attr_should_poll = False` explicitly — coordinator-based
  entities already inherit that from `CoordinatorEntity`.

## Async patterns

- Never call blocking I/O in a coroutine. Wrap sync SDK calls with
  `await hass.async_add_executor_job(fn, *args)`.
- Never `time.sleep()` in a coroutine — use `await asyncio.sleep(...)`.
- Use `hass.async_create_task(coro)` over a bare `asyncio.create_task(coro)`
  (HA tracks the task so it survives reload/teardown).
- Use `homeassistant.util.dt.utcnow()` over `datetime.now(UTC)`. The HA helper
  is patchable in tests.

## Imports

Every module starts with `from __future__ import annotations`.

- Same-package relative imports (`from .module import …`).
- Type-only imports inside `if TYPE_CHECKING:` block.
- `noqa` comments reserved for unavoidable framework constraints (e.g.
  `ARG001` for HA callback parameters).

## Docstrings

Style: PEP 257 with the **D211 + D213** pair (which is what `ignore = ["D203", "D212"]` selects in ruff).

- Every public class, function, method (including `@property`), and
  `__init__` has a docstring (Ruff D102/D107).
- A single-line docstring is preferred:
  ```python
  def foo() -> int:
      """Return the current count."""
  ```
- A multi-line docstring puts the summary on the **second** line (D213):
  ```python
  def foo() -> int:
      """
      Return the current count.

      The count resets at midnight UTC.
      """
  ```
- Class docstrings start immediately after the `class` line (D211 — no
  blank line between).
- Module-level docstring at the top of every `.py` file.

## Comments vs docstrings

- **Docstrings document the public contract** — what callers can rely on.
  Always required on public APIs.
- **Comments document the *why*** behind a non-obvious choice in the
  implementation. Default to writing none.
- Never describe *what* the code does in a comment — well-named identifiers
  already do that.
- No section dividers (`# --- section ---`).

## Logging

- Use the package-level `LOGGER` from `const.py`
  (`LOGGER: Logger = getLogger(__package__)`).
- Lazy `%`-formatting, never f-strings:
  ```python
  LOGGER.warning("Refresh failed: %s", exception)   # correct
  LOGGER.warning(f"Refresh failed: {exception}")     # wrong
  ```
- Levels: `debug` (poll summaries), `info` (lifecycle), `warning`
  (recoverable), `error`/`exception` (unrecoverable).
- Use `LOGGER.exception("msg")` inside `except:` blocks — it logs the
  traceback automatically. Don't use `LOGGER.error(..., exc_info=True)`
  when `.exception(...)` will do.
- Never log secrets.

## Error messages

Format: `"Failed to <verb> <object>: <cause>"`. Keep them short and
grep-able. Pre-validate inputs before network calls.

```python
raise <Domain>ApiClientError(
    f"Failed to fetch device status: {response.status}"
)
```

## Python version notes

- On `requires-python >= 3.14`, the parenthesis-free multi-type `except`
  (PEP 758, e.g. `except ValueError, TypeError:`) is valid syntax. Do **not**
  flag it as a bug or "Python 2 leftover" — it is equivalent to
  `except (ValueError, TypeError):`.

## Conventional commits

All repos use Conventional Commits. Release-please parses them for version
bumps and CHANGELOG generation.

| Type | Bump | Notes |
|---|---|---|
| `feat` | minor | new user-visible functionality |
| `fix` | patch | bug fix |
| `perf` | patch | runtime performance |
| `deps` | patch | dependency bumps — not in the Conventional Commits spec but recognised by release-please / Dependabot |
| `refactor` | none | no behaviour change |
| `test` | none | test-only changes |
| `docs` | none | docs only |
| `build` | none | build system / packaging |
| `ci` | none | CI/CD config |
| `style` | none | formatting only |
| `chore` | none | catch-all maintenance |

Subject line: imperative mood, lowercase, no trailing period.
Use scopes when useful: `fix(sensor): handle None battery value`.
`BREAKING CHANGE:` footer bumps major (works with any type).

## pyproject.toml standard

This is the **shape** to copy. Pin exactly and bump via Dependabot / Renovate
— do not copy the literal version strings below, set them to whatever is
current when you scaffold.

```toml
[project]
# Align with the python_requires of your homeassistant pin below.
requires-python = ">=<X.Y>"

[tool.uv]
package = false

[dependency-groups]
# Two groups so CI can install lint-only on lint jobs (faster) and dev on
# test jobs.
dev = [
    "homeassistant==<current>",
    "pre-commit==<current>",
    "pytest==<current>",
    "pytest-asyncio==<current>",
    "pytest-cov==<current>",
    "pytest-homeassistant-custom-component==<current>",
]
lint = [
    "ruff==<current>",
    "mypy==<current>",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
pythonpath = ["."]
addopts = [
    "--cov=custom_components/<domain>",
    "--cov-report=term-missing",
    "--cov-fail-under=90",
]

[tool.mypy]
python_version = "<X.Y>"           # must match requires-python
files = ["custom_components/<domain>"]
ignore_missing_imports = true
follow_imports = "silent"
strict_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_unreachable = true
no_implicit_optional = true
check_untyped_defs = true
disallow_untyped_defs = true
disallow_any_generics = true

# Relax inside tests so fixtures/mocks stay readable.
[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false
check_untyped_defs = false

[tool.ruff]
target-version = "py<XY>"          # must match python_version above

[tool.ruff.lint]
select = ["ALL"]
ignore = [
    "ANN401",  # `typing.Any` — `mypy --strict` rejects it more precisely
    "D203",    # no-blank-line-before-class (incompatible with formatter)
    "D212",    # multi-line-summary-first-line — we use D213 instead
    "COM812",  # incompatible with `ruff format`
    "ISC001",  # incompatible with `ruff format`
]

[tool.ruff.lint.flake8-pytest-style]
fixture-parentheses = false

[tool.ruff.lint.pyupgrade]
keep-runtime-typing = true

[tool.ruff.lint.mccabe]
max-complexity = 25

[tool.ruff.lint.per-file-ignores]
"tests/**" = [
    "S101",     # assert is normal in pytest
    "S105",     # hardcoded-password-string is fine for fake test credentials
    "S106",     # hardcoded-password-func-arg is fine for fake test credentials
    "D",        # no docstrings required in tests
    "ANN",      # no type annotations required in tests
    "ARG001",   # fixtures are used implicitly by pytest
    "PLC0415",  # local imports inside fixtures are intentional
    "SLF001",   # testing private HA attrs is intentional
    "PLR2004",  # magic numbers in test assertions are fine
]
```

## Pre-commit

`.pre-commit-config.yaml` mirrors the lint commands (`ruff format`,
`ruff check --fix`, `mypy`) and adds repo-hygiene hooks. As with
`pyproject.toml`, copy the **shape**, not the literal `rev` strings — bump revs
via `pre-commit autoupdate`.

```yaml
# .pre-commit-config.yaml
default_install_hook_types: [pre-commit]
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v<current>
    hooks:
      - id: ruff-format
      - id: ruff-check
        args: [--fix]

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v<current>
    hooks:
      - id: mypy
        additional_dependencies:
          - homeassistant==<current>
          # add SDK and any types-* packages mypy needs

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v<current>
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-toml
      - id: check-merge-conflict
      - id: mixed-line-ending
        args: [--fix=lf]
```
