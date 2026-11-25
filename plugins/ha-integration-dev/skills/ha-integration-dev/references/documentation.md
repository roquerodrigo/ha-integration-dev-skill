# Documentation: README, CONTRIBUTING and repository metadata

The README of an integration, theme or card repository is read in two places,
by two different readers:

| Where                                      | Reader                                             | What they need                                         |
|--------------------------------------------|----------------------------------------------------|--------------------------------------------------------|
| GitHub                                     | A user deciding to install, and contributors       | Everything                                             |
| Home Assistant Marketplace (HA 2026.11+)   | A user inside Home Assistant, one click from Install | What it does, what it needs, how to set it up and use it |

## Leaving sections out of the Marketplace

Wrap the GitHub-only parts in the Marketplace exclusion markers. GitHub hides
HTML comments, so the GitHub view keeps every section:

```markdown
<!-- ha-marketplace:exclude-start -->
...GitHub-only content...
<!-- ha-marketplace:exclude-end -->
```

How the Marketplace processes the README (`home-assistant/frontend`,
`src/panels/marketplace/tools/markdown.ts`):

- A section runs from a start marker to the next end marker. An unclosed one
  runs to the end of the file. Markers inside code are plain text.
- **Headings get no ids**, so an in-page link such as
  `[Configuration](#configuration)` leads nowhere in the Marketplace. Visible
  text never depends on one; name the section in words. Anchor links are
  fine inside excluded blocks.
- Relative links to `.md` files open on GitHub, relative images load from
  `raw.githubusercontent.com` at the version shown. Use relative paths for
  repository files.
- `#123` turns into a link to that issue.
- The Marketplace reads the README of the released version, so a README change
  only shows there after a release.

Hidden in the Marketplace: the badge block, **Installation** (the page has an
Install button), **Contributing** and **License**. **Support** (the sponsor
call) stays visible, because it speaks to users.

## Section order

Only the sections that apply, in this order. A repository whose `hacs.json`
declares a `country` writes the headings and all prose in that country's
language (pt-BR headings in brackets).

```
# <hacs.json name>                                 no "for Home Assistant"
<!-- exclude-start --> badges + open in Home Assistant button <!-- exclude-end -->
Lead paragraph                                     what the user gets; local or cloud; companion SDK link
(screenshot: required for cards and themes, light and dark for themes)

## Supported devices      [Dispositivos suportados]   model table, when there is more than one model
## Features               [Recursos]
## Requirements           [Requisitos]                HA version (= hacs.json), hardware, accounts, network
<!-- exclude-start -->
## Installation           [Instalação]
<!-- exclude-end -->
## Configuration          [Configuração]              UI path, fields table, Reconfigure and reauthentication
## Options                [Opções]
## Entities               [Entidades]                 one table per device kind
## Actions                [Ações]                     one ### per action, fields table, YAML example
## Events                 [Eventos]                   event types, payload, automation example
## Dashboard card         [Card de dashboard]         bundled card: YAML and options table
## How it works           [Como funciona]             data source, update interval or push, local vs cloud
## Troubleshooting        [Solução de problemas]
## Known limitations      [Limitações conhecidas]
## Support                [Apoie o projeto]
<!-- exclude-start -->
## Contributing           [Contribuindo]              two lines pointing to CONTRIBUTING.md and CODE_STYLE.md
## License                [Licença]
<!-- exclude-end -->
```

A card replaces Configuration, Options and Entities with **Usage** [Uso]
(minimal YAML, visual editor) and **Options** [Opções] (its YAML keys).

## Header

```markdown
# <Name>

<!-- ha-marketplace:exclude-start -->
[![CI](https://github.com/<owner>/<repo>/actions/workflows/ci.yml/badge.svg)](https://github.com/<owner>/<repo>/actions/workflows/ci.yml)
[![GitHub release](https://img.shields.io/github/v/release/<owner>/<repo>)](https://github.com/<owner>/<repo>/releases/latest)

[![Open your Home Assistant instance and open a repository in the Marketplace.](https://my.home-assistant.io/badges/marketplace_repository.svg)](https://my.home-assistant.io/redirect/hacs_repository/?owner=<owner>&repository=<repo>&category=<integration|plugin|theme>)
<!-- ha-marketplace:exclude-end -->
```

- The button shows the Marketplace badge but uses the `hacs_repository`
  redirect, which opens HACS before 2026.11 and the Marketplace from 2026.11
  on. A repository whose `hacs.json` requires 2026.11 or later may use the
  `marketplace_repository` redirect.
- `category=plugin` for a card, `category=integration` for an integration.
- Keep a sponsor badge if the repository has one. No "HACS Custom" shields
  badge and no `---` separator after the header.
- **A private repository gets no button**: `my.home-assistant.io` resolves the
  target through the public GitHub API.

## Installation

```markdown
## Installation

### Home Assistant Marketplace (recommended)

1. Select the **Open your Home Assistant instance** button above, or go to
   **Settings → Marketplace**, select **Add from link** in the upper right
   corner, enter `<owner>/<repo>` and select **Add**.
2. Select **Install**.
3. Restart Home Assistant.

The Marketplace comes with Home Assistant 2026.11 and later. On older
versions, add this repository to HACS as a custom repository of type
**Integration** and install it from there.

### Manual

1. Download `<domain>.zip` from the [latest release](https://github.com/<owner>/<repo>/releases/latest).
2. Extract it into `config/custom_components/<domain>/`.
3. Restart Home Assistant.
```

Drop the HACS paragraph when `hacs.json` requires 2026.11 or later. A card's
manual install copies the file to `config/www/` and adds the dashboard
resource.

## Configuration

No config-flow (`config_flow_start`) button. Give the UI path
(**Settings → Devices & services → Add integration**), a fields table whose
labels match `translations/en.json` exactly, ranges in words ("from 10 to
600"), and what Reconfigure and reauthentication do.

## Entities

Tables, not prose. Names exactly as the translations show them. Say which are
diagnostic or disabled by default.

## Writing

The reader is a Home Assistant user deciding whether this fits their home,
then setting it up. Describe the result for them, never the brief that
produced the code.

- Lead with what the user gets ("See who is connected to your mesh"), not
  with how it is built ("Custom integration that connects Home Assistant to
  the router's local web API"). Architecture words (coordinator, snapshot,
  registry, SDK internals, constants) stay out of the visible part; keep only
  what changes what the user sees or does.
- Rewrite rather than patch prose that restates requirements ("for each X the
  integration creates"), lists internal rules, or explains implementation
  choices.
- Plain words, short sentences, second person, present tense.
- Avoid the usual generated-text habits:
  - no em dash (—) or en dash (–), ranges included;
  - no bold lead-in on every bullet; bold is for UI labels and
    troubleshooting symptoms;
  - no "No X required", "out of the box", "seamless", "effortless",
    "robust", "leverage", "simply", "just", "Note that", "Whether you";
  - no forced groups of three, no emoji, no semicolon chains.
- Troubleshooting: the symptom in bold as the user would say it, then cause
  and fix in one or two sentences.
- Every statement is checked against the code: entities against the platform
  files and translations, defaults and ranges against `const.py`, versions
  against `hacs.json` and `manifest.json`.
- Public repository: nothing about the maintainer's own setup or tooling.

## Screenshots

A screenshot is public too, so it shows the real UI with made-up data. Nothing
from the maintainer's home goes in: no real device, entity, area, person or
network names, no real values that identify a home.

- Names are generic and in the language of the repository's prose (English,
  or the language of the `country` in `hacs.json`): "Front Door", "Robot
  Vacuum", "Leak Sensor", never the names on the instance the shot was taken
  from.
- The card or page also runs in that language, so its own labels match the
  names around them.
- Pick values that show every state the README describes (a low, a warning
  and a full battery, an unavailable entity), not whatever the home happens to
  report.
- For a card, render it on a live dashboard with a fabricated `hass`: start
  from the real object so the frontend keeps formatting and theming, and
  replace `states`, `entities`, `devices` and the language. A sketch for
  Playwright:
  ```js
  await page.locator("my-card").evaluate((card, states) => {
    const real = card.hass ?? card._hass;
    const fake = Object.create(real);
    Object.defineProperties(fake, {
      states: { value: states },
      entities: { value: {} },
      devices: { value: {} },
      locale: { value: { ...real.locale, language: "en" } },
      language: { value: "en" },
    });
    card.setConfig({ /* the options the README shows */ });
    card.hass = fake;
    // Stop the dashboard from pushing the real hass back on the next update.
    Object.defineProperty(card, "hass", { set() {}, configurable: true });
  }, fabricatedStates);
  await page.locator("my-card").screenshot({ path: "assets/preview.png" });
  ```
- Take the shot at `deviceScaleFactor: 2`, of the element alone rather than
  the whole page, and a phone width shot at about 390 px when the layout
  changes there.
- A shot that once showed real names is replaced in the same branch before it
  merges, and the branch is squashed so the old image does not stay in the
  history.

## CONTRIBUTING.md

Owns the developer content that leaves the README: setup (`scripts/setup`), a
dev instance (`scripts/develop`), the exact lint, format, type-check and test
commands (with the coverage gate from `pyproject.toml`), pre-commit,
Conventional Commits, translations and the pull request template. Replace the
old template text that mentions `black`.

## Repository description and topics

The Marketplace catalog requires both.

- Description: one or two plain sentences in the README's language, saying
  what the user gets, without dashes.
- Topics: `home-assistant`, `homeassistant`, `hacs`, plus
  `home-assistant-integration`, `home-assistant-card` or
  `home-assistant-theme`, then brand, product and use case
  (`presence-detection`). Keep existing relevant ones.

```sh
gh repo edit <owner>/<repo> --description "..." --add-topic home-assistant-integration
```
