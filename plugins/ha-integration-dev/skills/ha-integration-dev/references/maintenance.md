# Maintenance and closing flow

What "done" means after lint and tests pass: dependency upkeep and how a
change actually lands.

## Closing flow (public repos)

These repos protect `main` with required CI checks and accept changes only
through pull requests:

- **Start from fresh `main`**: update the local `main` from the remote before
  branching for any new work.
- **Every change goes through a PR** — never push to `main` directly, and
  never loosen branch protection to rush a fix through.
- **Merges are rebase-merges only** (merge commits and squash are disabled).
  Keep each commit a valid Conventional Commit: after a rebase-merge the
  individual commits land on `main` as-is, and release-please reads them one
  by one.
- **Opening the PR is where autonomous work stops.** Merging and releasing
  are the maintainer's call — leave the PR open for review instead of
  self-merging, unless explicitly told otherwise.
- PR titles and descriptions are formal and describe the change and its
  motivation — not the process that produced it.

## Validating on a live Home Assistant

Lint and tests are necessary, not sufficient — before a release, prove the
behaviour on a running Home Assistant instance:

1. Copy the changed integration into the instance's
   `config/custom_components/<domain>/`.
2. **Restart Home Assistant.** Reloading the config entry does **not**
   re-import Python modules — only a restart loads new code.
3. Confirm the change with evidence (logs, entity states, the UI), not
   assumption.

For an SDK change, build and install the wheel into the instance's
environment first — see [sdk.md](./sdk.md).

## Paired Home Assistant dependency bump

`homeassistant` and `pytest-homeassistant-custom-component` move **together**
— each release of the test harness targets one specific HA version. Bumping
one without the other produces confusing resolution failures or a harness
that tests against a different HA than the one pinned.

Procedure:

1. Pick the target `homeassistant` version.
2. Find the `pytest-homeassistant-custom-component` release built for it —
   check the harness release notes or its `requires_dist` metadata
   (`pip index` / PyPI JSON) for the matching `homeassistant==` requirement.
3. Bump both in the `dev` dependency group, run `uv sync`, and run the full
   gate (new HA versions regularly add deprecation warnings and behaviour
   changes the suite will surface).
4. Raise the `homeassistant` floor in `hacs.json` **only** when the
   integration now relies on an API the older HA lacks — the floor is a
   user-facing compatibility promise, not a mirror of the dev pin.

Keep the pin reasonably current. The `homeassistant==` pin fixes a large set
of transitive dependencies exactly (aiohttp, cryptography, ...), so a stale
pin blocks Dependabot security updates across the whole lock — the
characteristic symptom is Dependabot security jobs failing with
`dependency_file_not_resolvable`.
