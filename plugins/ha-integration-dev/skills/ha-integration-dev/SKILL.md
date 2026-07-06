---
name: ha-integration-dev
description: >-
  This skill should be used when working on a Home Assistant custom integration
  (a repo with a `custom_components/<domain>/` directory that follows the
  blueprint conventions) or a companion Python SDK that wraps a device or cloud
  API and is consumed by such an integration. Use it when the user asks to add
  entities, fix bugs, refactor,
  add tests, review code, scaffold a new integration from a blueprint, bump
  dependencies, or apply any code change in such a repo. Also use when the user
  mentions HACS, config flow, coordinator, reauth, translations, quality scale,
  or any Home Assistant integration development concept. It also covers building
  a Home Assistant **Lovelace custom card** — a zero-build frontend web component
  shipped as a HACS plugin/dashboard resource (the card `.js` lives at the repo
  root with a `hacs.json`, not in `custom_components/`). Use it too when the user
  asks to create or change a custom card, its `ha-form` visual editor,
  `window.customCards` registration, card i18n, or the card's HACS/CI/release
  flow.
version: 1.1.0
---

# Home Assistant Integration Development

Conventions and patterns for building Home Assistant custom integrations that
target **Platinum** on the HA Integration Quality Scale. The patterns assume a
shared blueprint (e.g. `ha-integration-blueprint`) as the starting point for
each new integration.

## When NOT to use

Skip this skill when working on:

- **Home Assistant core integrations** (the `home-assistant/core` repo).
  Core has its own conventions, review process, and constraints — applying
  this skill there will fight the upstream style.
- **Third-party custom integrations** authored by someone else that visibly
  do not follow the blueprint layout (no `runtime_data`, no
  `DataUpdateCoordinator` typing, no `CODE_STYLE.md`). Match the host repo's
  existing style instead.
- **Generic Python projects** that happen to use words like "coordinator" or
  "config flow" outside the HA context.
- **HA YAML configuration, automations, and user-side dashboards.** Editing a
  user's dashboard/automation YAML is out of scope. Building a distributable
  Lovelace **custom card**, however, *is* in scope — see
  [Lovelace custom cards](#lovelace-custom-cards).
- **Standalone SDKs / API clients with no HA integration consuming them.** The
  companion-SDK case applies only when a `custom_components/<domain>/`
  integration actually imports the package. A device/cloud API client that no
  HA integration depends on is out of scope — match its own house style.
- **MCP servers and other generic API wrappers.** Talking to a device or cloud
  API does not by itself bring a repo into scope; without an HA integration
  consumer, skip the skill.
- **TypeScript / non-Python repos** — *except* a Lovelace **custom card** (a JS
  frontend plugin), which this skill covers via
  `references/lovelace-cards.md`. The Python-only conventions and gates below
  (`ruff`/`mypy`/`pytest`, "tests ship with code") apply to integrations and
  SDKs, **not** to cards; cards have their own verification in that reference.

## Tests ship with code

Production code and its tests are a single deliverable — never commit or declare
done without both. When adding a feature, fixing a bug, or scaffolding a new
integration, write the corresponding tests in the same pass. "Write tests later"
is not acceptable; the verification workflow below will fail without them.

## Verification workflow

After every code change, run lint then tests before declaring done. Run the
tools **directly** — do not rely on a `scripts/lint` wrapper (the convention is
to drop that wrapper and invoke the underlying commands):

```bash
# Python repos — prefix each with `uv run` when the repo has a uv.lock:
ruff format --check .
ruff check .
mypy <path>            # custom_components/<domain> for integrations, src/ for SDKs
pytest
```

- In a uv-managed repo the lint tools live in the `lint` dependency group and
  are not on `PATH`; run via `uv run --group lint …` (and `uv sync` / an
  editable install first so `pytest` can import a `src/`-layout package).
- For a **read-only review**, use the non-mutating variants above
  (`ruff format --check`, `ruff check` without `--fix`) so you never modify the
  tree just to pass the gate.
- Both gates should mirror CI. Skip only for README-only edits.
- **Lovelace custom cards have no Python gate.** They are frontend JS — verify
  them with the card-specific steps in `references/lovelace-cards.md`
  (`node --check`, resource-served check, browser check), not `ruff`/`pytest`.

## Always read the repo's style guide first

If the repo has a `CODE_STYLE.md`, `CLAUDE.md`, or `AGENTS.md`, treat it as the
single source of truth for conventions. Read it before adding or restructuring
any file/class/function, and let its rules **override** this skill's defaults
where they conflict — a repo may deliberately diverge (e.g. on commit/string
language or payload typing). This skill summarises cross-integration patterns;
per-repo specifics live in that guide. Before trusting a stale-looking claim in
it (a referenced config file, a coverage percentage), cross-check against the
repo's actual `pyproject.toml` / CI and flag the drift.

---

## Architecture

Read `references/architecture.md` for the full data-flow, coordinator, config
flow, and entity design patterns.

## Coding conventions

Read `references/coding-conventions.md` for typing, naming, imports, properties,
logging, error handling, docstrings, and comment rules.

## Testing patterns

Read `references/testing.md` for the test structure, fixtures, mocking strategy,
coverage gates, and CI/CD pipeline.

## Code review

Read `references/code-review.md` for the per-area checklist to use when
reviewing a PR against a repo that follows these conventions.

## Creating a new integration

Read `references/new-integration.md` for the step-by-step checklist to fork a
blueprint into a new integration.

## Lovelace custom cards

Read `references/lovelace-cards.md` when the deliverable is a **frontend custom
card** rather than a Python integration — a zero-build web component shipped as
a HACS plugin/dashboard resource. It covers card-vs-integration, repo layout,
the web-component contract (`setConfig`/`set hass`/`getConfigElement`/
`customCards`), the `ha-form` visual editor, internationalization, manual vs.
HACS install, and the CI/release flow. The Python conventions, testing, and
quality-scale sections above do **not** apply to cards.
