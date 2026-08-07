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
version: 1.2.0
---

# Home Assistant Integration Development

Conventions and patterns for building Home Assistant custom integrations that
apply the pertinent **Bronze/Silver/Gold** rules of the HA Integration Quality
Scale — Platinum is an aspiration, not a review bar. The patterns assume a
shared blueprint (e.g. `ha-integration-blueprint`) as the starting point for
each new integration.

## When NOT to use

Skip this skill for:

- **`home-assistant/core` integrations** — core has its own conventions and
  review process; this skill will fight the upstream style.
- **Someone else's custom integration that doesn't follow the blueprint**
  (no `runtime_data`, no typed coordinator, no `CODE_STYLE.md`) — match the
  host repo's existing style instead.
- **Repos with no HA integration in the loop** — generic Python projects that
  merely use words like "coordinator", standalone SDKs/API clients that no
  `custom_components/<domain>/` integration imports, MCP servers and other
  API wrappers. Talking to a device API does not by itself bring a repo into
  scope.
- **User-side YAML** (automations, dashboards, configuration.yaml). Building a
  distributable Lovelace **custom card**, however, *is* in scope — see
  [Lovelace custom cards](#lovelace-custom-cards).
- **Other TypeScript / non-Python repos.** The Python gates below apply to
  integrations and SDKs, not to cards; cards have their own verification in
  `references/lovelace-cards.md`.

## Tests ship with code

Production code and its tests are a single deliverable — never commit or declare
done without both. When adding a feature, fixing a bug, or scaffolding a new
integration, write the corresponding tests in the same pass. "Write tests later"
is not acceptable; the verification workflow below will fail without them.

## Verification workflow

After every code change, run lint then tests before declaring done. The
standard is a thin `scripts/lint` wrapper that only chains the four commands
below (single source for CI, docs, and local runs) — use it when the repo has
one, or run the commands directly:

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

When the repo's guide is complete (the blueprint's `CODE_STYLE.md` +
`CLAUDE.md` already cover typing, layout, entity rules, and the verification
workflow), work from it directly and open this skill's references only for
what the guide does **not** cover — scaffolding a new repo, release wiring,
the review checklist, card development. Re-reading references that restate
the repo guide costs context and adds nothing.

## When you cannot ask the user

Some steps genuinely need user input (brand artwork, credentials, naming
choices). In an autonomous run where asking is impossible, never silently
substitute an invention — make the gap impossible to miss instead: record a
tracked TODO in the deliverable itself (README or issue), state it prominently
in your final report, and treat it as a release blocker. For example,
blueprint brand placeholders may stay as a stopgap so validation passes, but a
release that ships another project's artwork is a bug, not a shortcut.

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
