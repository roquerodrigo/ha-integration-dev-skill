---
name: ha-integration-dev
description: >-
  This skill should be used when working on a Home Assistant custom integration
  (any repo with a `custom_components/<domain>/` directory) or a companion
  Python/TypeScript SDK that wraps a device or cloud API for such an
  integration. Use it when the user asks to add entities, fix bugs, refactor,
  add tests, review code, scaffold a new integration from a blueprint, bump
  dependencies, or apply any code change in such a repo. Also use when the user
  mentions HACS, config flow, coordinator, reauth, translations, quality scale,
  or any Home Assistant integration development concept.
version: 1.0.0
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
- **HA YAML configuration / automations / dashboards.** This skill is for
  Python code in `custom_components/`, not user-side YAML.

## Verification workflow

After every code change, run lint then tests before declaring done:

```bash
scripts/lint && pytest
```

`scripts/lint` typically runs `ruff format`, `ruff check --fix`, and `mypy`.
Both gates should mirror CI. Skip only for README-only edits.

## Always read CODE_STYLE.md first

If the repo has a `CODE_STYLE.md`, treat it as the single source of truth for
conventions. Read it before adding or restructuring any file/class/function.
This skill summarises cross-integration patterns; per-repo specifics live in
`CODE_STYLE.md`.

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
