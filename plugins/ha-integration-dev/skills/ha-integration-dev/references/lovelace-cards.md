# Lovelace custom cards (frontend plugins)

A Lovelace **custom card** is a pure frontend web component — **not** a Python
integration. No `custom_components/`, no coordinator, no config flow, no
`manifest.json`. It ships as a single `.js` file registered as a Lovelace
**resource** and renders existing Home Assistant entities on a dashboard.

This reference covers the whole flow: deciding card vs. integration, repo
layout, the web-component contract, the visual editor, internationalization,
install/deploy (manual **and** HACS), CI/release, and verification.

Throughout, `<card-name>` is the kebab-case repo/element name (`my-card`),
`MyCard` is the PascalCase class, and `<Card Name>` is the human label.

## A card, not an integration

Ship a card — not Python — when the deliverable is **purely a dashboard
visual** over entities that already exist. If you are adding no new entities,
services, or device state, it is a card.

- **Rejected anti-pattern:** a `custom_components/<domain>` integration that
  **only** exists to serve a JS file via `add_extra_js_url`. That drags in a
  config entry, setup/unload, and HACS integration validation for what is
  really a frontend asset. If it renders, it's a card; if it fetches/owns
  state, it's an integration.
- The Python-only gates in this skill (`ruff`/`mypy`/`pytest`, "tests ship with
  code", quality scale) **do not apply** to cards. Cards have their own
  verification (below).

## Embedded companion card (inside an integration)

A real integration (one that owns entities/state) may legitimately **ship a
companion card** for its own entities instead of maintaining a second repo.
The anti-pattern above is an integration that serves *only* JS — not an
integration that also serves its card.

Layout and registration:

- The card lives at `custom_components/<domain>/www/<card-name>.js` and is
  served through HA's static path for the integration.
- Register it as a **Lovelace dashboard resource** (created/updated by the
  integration at setup, with the integration version as a `?v=` cache-buster)
  — **not** via `frontend.add_extra_js_url`. The `add_extra_js_url` route
  races the frontend at startup: a dashboard page loaded at boot can render
  before the extra module is injected, and the card never defines. The
  dashboard-resource route survives that race.
- Guard the definition for re-imports and double-registration:
  ```js
  if (!customElements.get("my-card")) {
    customElements.define("my-card", MyCard);
  }
  ```
  A second `customElements.define` for the same tag throws and kills the
  card, and an embedded card can be loaded twice during resource migrations.
- The card file itself follows every rule in this reference (contract,
  editor, i18n, escaping); the Python side follows the integration
  references, and `node --check` on the card joins the repo's verification.

## Repo layout (zero-build, `content_in_root`)

Default to a **single-file, zero-build** vanilla `HTMLElement` (no Lit, no
bundler). A build step is a deliberate later choice, not the starting point.

```
<card-name>.js                # the card, at repo root
hacs.json                     # HACS manifest (see below)
package.json                  # name, main: <card-name>.js, repo url
README.md                     # badges + screenshot + usage (see below)
LICENSE
release-please-config.json
.release-please-manifest.json
.github/workflows/ci.yml      # HACS validation + release-please
.github/{dependabot.yml,CODEOWNERS,ISSUE_TEMPLATE/}
```

## hacs.json

Only these keys are valid: `name` (**required**), `content_in_root`,
`filename`, `country`, `homeassistant`, `persistent_directory`,
`render_readme`, `zip_release`, `hide_default_branch`.

```json
{
  "name": "<Card Name>",
  "filename": "<card-name>.js",
  "content_in_root": true,
  "render_readme": true,
  "homeassistant": "2026.5.0"
}
```

> There is **no `hacs` key**. Adding `"hacs": "<version>"` fails the HACS
> `hacsjson` validation check ("invalid hacs.json"). `homeassistant` is the
> minimum HA version, not a HACS version.

## The card contract (web component)

A custom card is a web component implementing this contract:

```js
class MyCard extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: "open" });
    this._config = {};
    this._hass = null;
    this._signature = null; // render-skip guard
  }

  // Return the editor element (see "Visual editor").
  static getConfigElement() { return document.createElement("my-card-editor"); }

  // Default config used when the user adds the card from the picker.
  static getStubConfig() { return { type: "custom:my-card" }; }

  // Validate + store. Throw on bad input. Must be cheap and idempotent.
  setConfig(config) {
    if (config.threshold != null && typeof config.threshold !== "number") {
      throw new Error('my-card: "threshold" must be a number');
    }
    this._config = { threshold: config.threshold ?? 20, ...config };
    this._signature = null;              // force a re-render
    if (this._hass) this._render();
  }

  // Called on every state update. Store + render.
  set hass(hass) { this._hass = hass; this._render(); }

  getCardSize() { return 3; }
  getGridOptions() { return { min_columns: 6, min_rows: 3 }; }

  _render() {
    if (!this._hass) return;
    // Skip the rebuild when nothing visible changed — avoids flicker.
    const signature = JSON.stringify([/* visible state incl. language */]);
    if (signature === this._signature) return;
    this._signature = signature;

    this.shadowRoot.innerHTML = `
      <style>${MyCard.styles}</style>
      <ha-card>...</ha-card>`;

    // Tap → open the standard more-info dialog.
    this.shadowRoot.querySelectorAll(".row").forEach((el) =>
      el.addEventListener("click", () =>
        this.dispatchEvent(new CustomEvent("hass-more-info", {
          detail: { entityId: el.dataset.id }, bubbles: true, composed: true,
        }))));
  }

  static get styles() { return `/* use HA design tokens, see below */`; }
}

customElements.define("my-card", MyCard);

// Register in the card picker.
window.customCards = window.customCards || [];
window.customCards.push({
  type: "my-card",
  name: "<Card Name>",
  description: "What it shows.",
  preview: true,
  documentationURL: "https://github.com/<owner>/ha-<card-name>",
});
```

Rules:

- **Style with HA design tokens, never hardcoded colors.** Render inside
  `<ha-card>` and use CSS custom properties — `var(--primary-text-color)`,
  `var(--divider-color)`, `var(--error-color)`, `var(--warning-color)`,
  `var(--success-color)`, `var(--primary-color)`. The theme and light/dark mode
  then follow automatically.
- **`<ha-icon>` sizes itself** from `var(--mdc-icon-size, 24px)`. Don't override
  it unless you deliberately want a non-standard size.
- **Read `hass.states` / `hass.entities` / `hass.devices`** for auto-discovery;
  prefer the device name (`devices[id].name_by_user || .name`) over a raw
  entity's `friendly_name`.
- **Escape every dynamic string interpolated into `innerHTML`.** Entity
  names, friendly names, and config values (`title`, labels) are
  user-controlled: interpolating them raw breaks the markup and is an XSS
  vector (`<img onerror=…>` in a renamed entity executes). Route text through
  an escape helper — attribute values included:
  ```js
  const escapeHtml = (value) =>
    String(value).replace(/[&<>"']/g, (ch) => ({
      "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;",
    }[ch]));
  ```
  Only trusted, card-owned fragments (your own `<style>` block, icon names
  you validated) may skip it.

## Visual editor (`ha-form`)

The editor is its own custom element. Build an `ha-form` schema, set
`computeLabel`, and dispatch `config-changed`:

```js
class MyCardEditor extends HTMLElement {
  setConfig(config) { this._config = config; this._render(); }
  set hass(hass) { this._hass = hass; this._render(); }

  _schema() {
    return [
      { name: "title", selector: { text: {} } },
      { name: "mode", selector: { select: { mode: "dropdown", options: [
        { value: "all", label: "All" }, { value: "low", label: "Low" },
      ] } } },
      { name: "threshold", selector: { number: { min: 0, max: 100, mode: "box" } } },
    ];
  }

  _render() {
    if (!this._hass) return;
    if (!this._form) {
      this._form = document.createElement("ha-form");
      this._form.computeLabel = (s) => this._labels[s.name] ?? s.name;
      this._form.addEventListener("value-changed", (ev) => {
        // Round-trip the whole config: spread the existing config first so
        // keys the schema doesn't cover (hand-written YAML) survive edits.
        const config = { ...this._config, ...ev.detail.value };
        for (const [key, value] of Object.entries(config)) {
          if (value === "" || value == null) delete config[key];
        }
        config.type = this._config.type ?? "custom:my-card";
        this.dispatchEvent(new CustomEvent("config-changed", {
          detail: { config }, bubbles: true, composed: true }));
      });
      this.appendChild(this._form);
    }
    this._form.hass = this._hass;
    this._form.schema = this._schema();
    this._form.data = { title: this._config.title ?? "", /* ... */ };
  }
}
customElements.define("my-card-editor", MyCardEditor);
```

Keep option **values** stable (they're persisted config); only the **labels**
are display text and get localized.

Two editor rules that come straight from field bugs:

- **The editor must round-trip the full config.** Emitting only
  `ev.detail.value` silently **deletes** every key the `ha-form` schema
  doesn't model — a hand-written `entities:` list in YAML disappears the
  moment the user opens the visual editor. Always spread `this._config`
  first, then the form values (as in the example above).
- **Prune empty values instead of persisting them.** A cleared text field
  emits `""`; stored as `title: ""` it freezes the header blank and defeats
  any localized runtime default (which only applies when the key is absent).
  Delete empty-string/nullish keys before dispatching `config-changed`.

## Internationalization

A pure frontend card **cannot** use `hass.loadBackendTranslation` — that loads a
custom *integration*'s `translations/*.json`, and a card has no backend
component. Embed the dictionaries and localize yourself. This is the standard
community-card pattern (e.g. mushroom, device-card).

```js
const TRANSLATIONS = {
  en: { "card.all": "All", "editor.threshold": "Low battery threshold" },
  "pt-BR": { "card.all": "Todas", "editor.threshold": "Limite de bateria fraca" },
};

// Active HA UI language, with a supported fallback then English.
function resolveLang(hass) {
  const lang = (hass && (hass.locale?.language || hass.language || hass.selectedLanguage)) || "en";
  if (TRANSLATIONS[lang]) return lang;
  if (lang.split("-")[0] === "pt") return "pt-BR";
  return "en";
}

// Translate a key; English is the fallback, then the key itself.
function localize(hass, key) {
  const t = TRANSLATIONS[resolveLang(hass)];
  return t?.[key] ?? TRANSLATIONS.en[key] ?? key;
}
```

- Read the language from `hass.locale?.language` (fall back to `hass.language`);
  match exact code, then base language (`pt` → `pt-BR`), then `en`.
- Localize **both** the card UI **and** the editor — `computeLabel` and every
  select-option `label` go through `localize(this._hass, …)`.
- Default any user-facing default string (e.g. the card title) to `null` in
  `setConfig` and resolve it at render time via `localize`, so it follows the
  language instead of freezing one locale into the stored config.
- Include the resolved language in the render-skip **signature** so switching
  the HA language re-renders.

## Install & deploy

Two paths. They are **mutually exclusive** — never register the card from both
at once, or the second `customElements.define("my-card", …)` throws
`"my-card" has already been used` and the card fails to load.

### Manual (`www/` + resource)

1. Copy `<card-name>.js` to `config/www/`.
2. Register a Lovelace resource: URL `/local/<card-name>.js?v=N`, type
   **module**. Bump `?v=N` whenever you change the file to bust the browser
   cache.
3. Restart HA (needed the first time you add a `www/` file or change a
   resource).

### HACS (custom repository) — preferred for a public card

- Category **Dashboard** / `plugin`. Served from
  `/hacsfiles/<repo>/<card-name>.js`. Users add it via the **My Home Assistant**
  button (below); HACS then delivers updates on each release — no manual `cp` or
  `?v` bump.
- **Migrating manual → HACS:** remove the `/local/<card-name>.js` resource
  **and** the `config/www/<card-name>.js` copy, then restart HA, so only the
  `/hacsfiles/…` resource remains (otherwise the duplicate-registration
  conflict above).

## CI / release

CI reuses the shared workflows in `roquerodrigo/workflows` (same as the
integration repos), plus a syntax check on the card itself:

```yaml
# .github/workflows/ci.yml
jobs:
  syntax:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: node --check <card-name>.js

  validate:
    uses: roquerodrigo/workflows/.github/workflows/home-assistant-validate.yml@main
    with:
      hassfest: false        # a card has no manifest.json
      version-check: false   # nothing to compare against pyproject.toml
      hacs-category: plugin
```

The release lives in its own `release.yml`, gated on a successful CI run
(`workflow_run` on CI completion, `push` event only) and calling
`roquerodrigo/workflows/.github/workflows/release-please.yml@main` with
`release-token: ${{ secrets.RELEASE_PLEASE_PAT }}` — the same shape the
integration repos use (see `testing.md`, "CI/CD pipeline").

HACS `plugin` validation runs these checks: `license`, `information`, `topics`,
`archived`, `description`, `issues`, `hacsjson`, `images`. To go green:

- **`hacsjson`** — valid keys only (no `hacs` key).
- **`topics`** — the repo must have GitHub topics:
  `gh repo edit <owner>/ha-<card-name> --add-topic home-assistant,lovelace,lovelace-card,custom-card,hacs,dashboard`
- **`images`** — the README must contain at least one image. Add a **real
  screenshot** to `assets/` and reference it. Prefer a genuine screenshot over
  the `ignore: images` action input; `ignore` is a last resort for a card that
  is manual-install only and will never be in the HACS store.
- **release-please** needs GitHub Actions permitted to open PRs. Either enable
  it once —
  `gh api --method PUT repos/<owner>/ha-<card-name>/actions/permissions/workflow -F default_workflow_permissions=write -F can_approve_pull_request_reviews=true`
  — or supply a PAT. Without it the release job fails with *"GitHub Actions is
  not permitted to create or approve pull requests."* Prefer the
  `RELEASE_PLEASE_PAT` secret anyway: release PRs opened with the default
  `GITHUB_TOKEN` trigger no workflows and sit with stale checks.

Repo setup: **public**, topics added, description set, issues enabled.

## README essentials

The header is the same one the sibling integration repos use, in this exact
order: **title → badges → HACS link → `---` separator → the rest of the
document.**

```markdown
# <Card Name>

[![CI](https://github.com/<owner>/ha-<card-name>/actions/workflows/ci.yml/badge.svg)](https://github.com/<owner>/ha-<card-name>/actions/workflows/ci.yml)
[![hacs_badge](https://img.shields.io/badge/HACS-Custom-orange.svg)](https://github.com/hacs/integration)

[![Open your Home Assistant instance and open the repository inside HACS.](https://my.home-assistant.io/badges/hacs_repository.svg)](https://my.home-assistant.io/redirect/hacs_repository/?owner=<owner>&repository=ha-<card-name>&category=plugin)

---
```

- **`category=plugin`, not `integration`** — HACS distributes a card as a
  dashboard resource, so `category=integration` sends the button looking for a
  `custom_components/` payload this repository does not have.
- `repository=` is the repository name and `owner=` its GitHub owner.
- The HACS link is a paragraph of its own, separated from the badge block by a
  blank line — the badge row and the "Open your Home Assistant instance" button
  are different things and must not run together.
- Preserve whatever badges the repository already carries and invent no new
  ones; all of them belong **before** the HACS link.
- The `---` right after the HACS link is the header separator. A README that
  already uses `---` further down keeps those as section breaks — do not add a
  second separator to the header.
- **A private repository gets no HACS link.** `my.home-assistant.io` resolves
  the target through the public GitHub API, so on a private repository the
  button lands on something HACS cannot install. It is the same reason those
  repositories run validation with `hacs: false`. Ship the title, the badges
  and the separator, and add the link once the repository is public. Cards are
  public by default (HACS `plugin` validation requires it), so this normally
  only bites a card kept private while it is being built.

Then: a **screenshot** (also satisfies the HACS `images` check), a short feature
list, a `type: custom:my-card` usage block, and an options table. A card with a
visual editor should still document YAML — the editor is a convenience, not the
spec.

## Verification (no pytest here)

Cards have no Python gate. Instead:

- `node --check <card-name>.js` — syntax.
- Deploy, then confirm the resource serves: `curl -s -o /dev/null -w '%{http_code}'
  http://<ha-host>:8123/hacsfiles/ha-<card-name>/<card-name>.js` → `200` (and the
  old URL → `404` after a rename/migration).
- In the browser: hard-refresh (or bump `?v`), confirm there is **no** "Custom
  element doesn't exist" error, the console shows the card's `loaded` line, and
  the **visual editor opens** (exercises the editor element).

## Gotchas

- **Text truncation needs `nowrap` + a `title`.** `text-overflow: ellipsis`
  only applies with `white-space: nowrap` (+ `overflow: hidden` and a
  `min-width: 0` flex child). Add a `title="<full text>"` attribute for
  accessibility — `alt` is only valid on `<img>`, not on a `<span>`.
- **Responsive column cap in one line.** To cap at N columns on wide screens and
  degrade responsively, drive the grid from a CSS variable:
  `grid-template-columns: repeat(auto-fill, minmax(max(<min>px, calc((100% - (var(--cols) - 1) * <gap>px) / var(--cols))), 1fr));`
  and inject `style="--cols:${config.columns}"` on the grid.
- **Render-skip signature.** Serialize the *visible* state (mode, sort, title,
  language, the shown items + values) and bail when unchanged; otherwise every
  `hass` tick rebuilds the DOM and flickers.
- **`.storage/` ownership.** On the HA host, `lovelace_resources` and `www/` are
  usually user-owned, but `core.config_entries` is root-owned — editing it needs
  `sudo`.
