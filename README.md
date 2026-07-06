# ha-integration-dev-skill

A Claude Code plugin that teaches Claude the conventions, architecture, and
testing patterns for building **Home Assistant custom integrations** and their
companion Python SDKs.

The patterns target **Platinum** on the HA Integration Quality Scale and
assume a shared blueprint (e.g. `ha-integration-blueprint`) as the starting
point for each new integration.

## What's inside

A single skill — `ha-integration-dev` — that auto-loads when Claude Code is
working in a repo containing `custom_components/<domain>/` or in a related
Python SDK. The skill bundles:

- **Architecture** — file layout, runtime data, coordinator, config/options
  flow, entity design, exception hierarchy, translations, diagnostics, HACS.
- **Coding conventions** — strict typing, naming, imports, properties,
  logging, error messages, Conventional Commits, `pyproject.toml` template.
- **Testing patterns** — test layout, coverage gates, `conftest.py` pattern,
  mocking strategy, translation parity tests, CI/CD jobs.
- **Code review checklist** — per-area checklist for PR review.
- **New integration scaffolding** — step-by-step checklist to fork a
  blueprint into a fresh integration.

Reference files are loaded on demand by Claude — only what's relevant to the
current task ends up in context.

## Installation

In a Claude Code session:

```text
/plugin marketplace add roquerodrigo/ha-integration-dev-skill
/plugin install ha-integration-dev
```

After install, start a session inside any HA integration repo and the skill
loads automatically when its triggers match.

## When the skill triggers

- Repos containing `custom_components/<domain>/`
- Companion Python SDKs that wrap a device or cloud API for such an
  integration
- Conversations mentioning: HACS, config flow, coordinator, reauth,
  translations, quality scale, or other HA integration concepts

## When it does NOT trigger

The skill explicitly steps aside for:

- Home Assistant core development (`home-assistant/core`)
- Third-party integrations that visibly do not follow the blueprint layout
- HA YAML configuration, automations, and dashboards (user-side, not Python)

## Layout

```
.claude-plugin/marketplace.json
plugins/ha-integration-dev/
├── .claude-plugin/plugin.json
└── skills/ha-integration-dev/
    ├── SKILL.md
    └── references/
        ├── architecture.md
        ├── coding-conventions.md
        ├── testing.md
        ├── code-review.md
        └── new-integration.md
```

## License

MIT
