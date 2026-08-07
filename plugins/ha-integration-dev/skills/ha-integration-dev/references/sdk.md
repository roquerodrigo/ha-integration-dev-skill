# Companion SDK development

A companion SDK is a standalone Python package that wraps a device or cloud
API and is consumed by a Home Assistant integration through
`manifest.json` `requirements`. It lives in its own repository, releases to
PyPI, and knows nothing about Home Assistant — all HA-specific mapping
(entities, coordinators, `ConfigEntry` plumbing) stays in the integration.

## Repo layout

```
src/<package_name>/          # src layout — the package is importable only
│                            #   after an (editable) install, which keeps
│                            #   tests honest about packaging
├── __init__.py              # public surface re-exports
├── py.typed                 # PEP 561 marker — ships the type information
└── ...                      # one class per file, same rules as integrations
tests/                       # network-free by default; hardware/network
                             #   suites are opt-in (marker or env var)
pyproject.toml               # hatchling build backend
LICENSE                      # must exist — publishing "MIT" metadata without
                             #   the license text is a distribution bug
release-please-config.json
.release-please-manifest.json
.github/workflows/           # same shared reusables as the integrations
```

- Build backend: **hatchling**. Include `py.typed` so consumers get types.
- mypy runs with **`strict = true`** — an SDK owns its whole type surface,
  so there is no framework boundary to relax for.
- Set `readme` in `[project]` — without it the PyPI page renders empty.

## Dependency policy — never pin runtime dependencies

**Runtime dependencies (`[project] dependencies`) are never pinned with `==`
and never carry an upper bound. Use a `>=` floor only.**

```toml
[project]
dependencies = [
    "aiohttp>=3.11",       # correct — floor only
    # "aiohttp==3.11.13",  # wrong — conflicts inside Home Assistant
    # "aiohttp>=3.11,<4",  # wrong — the cap bites the moment HA moves first
]
```

The reason is the consumer: Home Assistant pins its own transitive
dependencies exactly (its `package_constraints.txt`). An SDK that pins — or
caps — a library HA also ships will sooner or later contradict HA's pin, and
the integration fails to install. Exact pins belong in dev/lint dependency
groups and lock files, which never reach the consumer.

Note the asymmetry with the **integration side**: the integration's
`manifest.json` pins the SDK **exactly** (`"<sdk-package>==<version>"`), so
what runs inside HA only ever changes through an explicit bump PR.

## Manifest × dev-pin parity

The integration tests run against the SDK version in its `pyproject.toml`
dev group, while HA installs the version pinned in `manifest.json`. When the
two drift, CI is green against a version users never run. Every integration
consuming an SDK should carry this test:

```python
def test_manifest_sdk_pin_matches_dev_group() -> None:
    """The SDK version HA installs is the one the test suite runs against."""
    manifest = json.loads(
        Path("custom_components/<domain>/manifest.json").read_text()
    )
    requirement = next(
        item for item in manifest["requirements"]
        if item.startswith("<sdk-package>")
    )
    pyproject = tomllib.loads(Path("pyproject.toml").read_text())
    assert requirement in pyproject["dependency-groups"]["dev"]
```

## Release and publish flow

The `release.yml` composes three shared reusables, gated on a successful CI
run (`workflow_run`, `push` event only):

1. **`release-please.yml@main`** grooms the release PR and tags the version
   on merge (`release-token: ${{ secrets.RELEASE_PLEASE_PAT }}`).
2. **`sync-uv-lock.yml@main`** refreshes `uv.lock` on the release branch so
   the lock lands carrying the released version.
3. **`publish-pypi.yml@main`** builds and publishes the tag to PyPI
   (`package:` name, `pypi-token: ${{ secrets.PYPI_API_TOKEN }}`); a
   `workflow_dispatch` input allows republishing an existing tag.

## SDK fix → integration bump

A fix that must reach Home Assistant travels in three steps, in order:

1. **PR on the SDK** (fix + tests), merge, let release-please cut the release
   and publish to PyPI.
2. **Bump PR on the integration**: update the `==` pin in `manifest.json`
   `requirements` *and* the dev-group pin in `pyproject.toml` (the parity test
   above keeps them honest), plus any code change the new SDK version needs.
3. Merge the bump PR; the integration's own release follows.

## Validate with a local wheel before releasing

Never let the first real-world execution of an SDK change be the released
version. Before merging the SDK's release PR:

```bash
uv build                                   # produces dist/<pkg>-<ver>-py3-none-any.whl
pip install --force-reinstall dist/<pkg>-*.whl   # into the consuming environment
```

Install the wheel into the environment where the integration actually runs
(a live HA instance's Python environment, or the integration's dev venv) and
exercise the changed behaviour A/B before and after. Remember that reloading
a config entry does **not** re-import Python modules — restart HA to load the
new wheel.
