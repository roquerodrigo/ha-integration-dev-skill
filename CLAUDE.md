# ha-integration-dev-skill

Claude Code **plugin marketplace repo**, not an application. It ships one
plugin (`ha-integration-dev`) containing one skill of the same name. The
skill teaches Claude the conventions for building Home Assistant custom
integrations, their companion Python SDKs, and Lovelace custom cards. There
is no source code to compile or run here — the deliverable is Markdown +
JSON that Claude Code loads at skill-invocation time.

## Skill activation

`plugins/ha-integration-dev/skills/ha-integration-dev/references/*.md` is
loaded on demand by Claude, not preloaded — keep `SKILL.md` terse (see
"Content conventions" below). `SKILL.md`'s `description:` frontmatter field
is what Claude Code's skill matcher scans to decide when to auto-load this
skill — keep it specific (trigger repo shapes, keywords) since that text, not
the body, drives activation.

## There is nothing to build/lint/test

This repo has no `package.json`, no Python project, no CI workflow
(`.github/workflows/` does not exist). "Verifying" a change means:

1. Read the edited Markdown/JSON back and check it's internally consistent
   (cross-references between `SKILL.md` and `references/*.md` still resolve,
   no dangling section links).
2. If practical, actually exercise the skill: open a session in a real HA
   integration repo (e.g. a repo with `custom_components/<domain>/`) and
   confirm the skill still triggers and reads sensibly.

Do not go looking for a lint/test command — there isn't one.

## Versioning: three files, four declarations

The plugin version is declared four times across three files and Claude Code
does not enforce that they match:

- `.claude-plugin/marketplace.json` → `metadata.version` **and**
  `plugins[0].version`
- `plugins/ha-integration-dev/.claude-plugin/plugin.json` → `version`
- `plugins/ha-integration-dev/skills/ha-integration-dev/SKILL.md` →
  frontmatter `version`

When bumping the skill (new behavior, not just a typo fix), bump all four
declarations to the same value in a single commit. There is no release
automation and no git tags — the version only ever moves because someone
edits these files. A bump that touches one file and not the others is a bug,
so read all four values back after editing and confirm they agree:

```sh
grep -rn '"version"' .claude-plugin/marketplace.json \
  plugins/ha-integration-dev/.claude-plugin/plugin.json
grep -n '^version:' plugins/ha-integration-dev/skills/ha-integration-dev/SKILL.md
```

## Content conventions (for editing SKILL.md / references)

- The skill applies the pertinent **Bronze/Silver/Gold** rules of the HA
  Integration Quality Scale (Platinum is an aspiration, not a review bar) and
  assumes every target repo forked a shared blueprint
  (`ha-integration-blueprint`) — don't add advice that only applies to
  unstructured/non-blueprint integrations.
- `SKILL.md` is deliberately terse; anything long or situational belongs in
  a `references/*.md` file loaded on demand, not inlined in the always-loaded
  body.
- The skill explicitly defers to a target repo's own `CODE_STYLE.md` /
  `CLAUDE.md` / `AGENTS.md` when present and complete — don't duplicate
  content that a repo's own guide already covers; the skill is meant to fill
  gaps, not restate them.
- Lovelace custom cards are explicitly **out of** the Python gates
  (`ruff`/`mypy`/`pytest`) — they're plain JS with their own verification in
  `lovelace-cards.md`. Don't accidentally imply cards need the Python
  toolchain.

## Git / PR flow

This is a **public** GitHub repo (`roquerodrigo/ha-integration-dev-skill`)
with feature branches and PRs for every change (see `git log` — commits are
Conventional Commits, PRs referenced as `(#1)`, `(#2)`, …) — this follows the
global public-repo convention (branch + PR, never push straight to `main`,
merge commits only). Update `main` locally before branching for a new
feature.
