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

This repo has no `package.json` and no Python project. The workflows are
`.github/workflows/auto-assign.yml` (issue/PR triage, delegated to
`roquerodrigo/workflows`) and `.github/workflows/release.yml` (Release
Autopilot) — there is no build/lint/test job. "Verifying" a change means:

1. Read the edited Markdown/JSON back and check it's internally consistent
   (cross-references between `SKILL.md` and `references/*.md` still resolve,
   no dangling section links).
2. If practical, actually exercise the skill: open a session in a real HA
   integration repo (e.g. a repo with `custom_components/<domain>/`) and
   confirm the skill still triggers and reads sensibly.

Do not go looking for a lint/test command — there isn't one.

## Versioning: Release Autopilot owns it

The plugin version is declared four times across three files and Claude Code
does not enforce that they match:

- `.claude-plugin/marketplace.json` → `metadata.version` **and**
  `plugins[0].version`
- `plugins/ha-integration-dev/.claude-plugin/plugin.json` → `version`
- `plugins/ha-integration-dev/skills/ha-integration-dev/SKILL.md` →
  frontmatter `version`

**Never edit them by hand.** `.github/workflows/release.yml` runs
Release Autopilot on every push to `main`; it keeps a release PR open, and that
PR bumps all four declarations (`version_files` in `.release-autopilot.yml`)
and `CHANGELOG.md` together. Merging the release PR cuts the `vX.Y.Z` tag and
the GitHub release. A feature PR that touches a version is a bug.

- None of these files is detected on its own, so each declaration needs its
  entry in `version_files`; the `SKILL.md` one is found by the
  `^version: (\S+)` pattern, so keep the `version:` line in that shape.
- **The commit type decides the version**, so pick it by what changes for
  someone using the skill, not by the fact that the file is Markdown: `feat:`
  for new or changed guidance (minor), `fix:` for guidance that was wrong
  (patch), `docs:` only for repo-facing docs (`README.md`, this file). A
  release PR only opens once the commits since the last tag hold something
  that matters to someone using the skill; a lone `docs:` or `ci:` commit
  waits, unreleased, until such a change lands.

After a release PR merges, confirm the four values still agree:

```sh
grep -rn '"version"' .claude-plugin/marketplace.json \
  plugins/ha-integration-dev/.claude-plugin/plugin.json
grep -n '^version:' plugins/ha-integration-dev/skills/ha-integration-dev/SKILL.md
```

## Content conventions (for editing SKILL.md / references)

- The skill assumes every target repo forked a shared blueprint
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
rebase merge only — merge commits and squash are disabled). Update `main`
locally before branching for a new feature.
