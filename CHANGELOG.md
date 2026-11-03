# Changelog

## [1.7.0](https://github.com/roquerodrigo/ha-integration-dev-skill/compare/v1.6.0...v1.7.0) (2026-11-03)

This release extends the developer guidance for Home Assistant integrations and Lovelace cards with form editors, automation triggers, setup error handling, a quality scale review and an index of official documentation.

### New features

- Developer guidance for Lovelace cards now covers a form editor generated from a schema, picker suggestions, and safer rendering of missing or unavailable entities. ([#31](https://github.com/roquerodrigo/ha-integration-dev-skill/pull/31))
- Developer guidance now covers integration-provided automation triggers and conditions built on the trigger and condition platforms, and flags legacy device automation files. ([#31](https://github.com/roquerodrigo/ha-integration-dev-skill/pull/31))
- Developer guidance now explains which setup exception to raise, separating retryable problems from credential and permanent errors, with retry delays and translated messages. ([#31](https://github.com/roquerodrigo/ha-integration-dev-skill/pull/31))
- Adds an index of official Home Assistant contributor documentation with instructions for fetching current pages instead of relying on memory. ([#31](https://github.com/roquerodrigo/ha-integration-dev-skill/pull/31))
- Adds a quality scale reference with the Bronze-to-Platinum rules as a review lens for integrations. ([#31](https://github.com/roquerodrigo/ha-integration-dev-skill/pull/31))

### Bug fixes

- Card examples no longer include the card type in the starting configuration, which Home Assistant adds automatically. ([#31](https://github.com/roquerodrigo/ha-integration-dev-skill/pull/31))

## [1.6.0](https://github.com/roquerodrigo/ha-integration-dev-skill/compare/v1.5.1...v1.6.0) (2026-11-03)

### Bug fixes

- Projects set up with the skill now validate and publish with Home Assistant Autopilot, so their CI and releases no longer fail on retired workflows. ([#29](https://github.com/roquerodrigo/ha-integration-dev-skill/pull/29))

## [1.5.1](https://github.com/roquerodrigo/ha-integration-dev-skill/compare/v1.5.0...v1.5.1) (2026-11-03)

### Improvements

- Release guidance now describes Release Autopilot, including the release pull request, the required secrets and how the first version is chosen. ([#26](https://github.com/roquerodrigo/ha-integration-dev-skill/pull/26))

### Bug fixes

- Projects no longer pin pytest and its plugins directly, since the Home Assistant test harness pins them, and Dependabot no longer proposes bumps that cannot resolve. ([#28](https://github.com/roquerodrigo/ha-integration-dev-skill/pull/28))

## [1.5.0](https://github.com/roquerodrigo/ha-integration-dev-skill/compare/v1.4.1...v1.5.0) (2026-09-25)


### Features

* keep debug logging on for every integration installed on a live instance ([900fe04](https://github.com/roquerodrigo/ha-integration-dev-skill/commit/900fe0460fb7399de53e6dbc8115d71ab2890c6e))

## [1.4.1](https://github.com/roquerodrigo/ha-integration-dev-skill/compare/v1.4.0...v1.4.1) (2026-09-18)


### Continuous Integration

* release with release-please ([9718a21](https://github.com/roquerodrigo/ha-integration-dev-skill/commit/9718a21ac96281ffee763d16cc6e5b392980456d))
