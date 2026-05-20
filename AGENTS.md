# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## Overview

Copenhagen Theme is the default Zendesk Guide (Help Center) theme. It's a Curlybars-based theme with React components for complex UI elements, built using Rollup and SCSS.
This theme is a custom Zendesk theme to be built using the base Copenhagen theme as a starting point for structure.

## Common Commands

```bash
# Development (builds + watches + starts local preview)
yarn start

# Build for production
yarn build

# Run tests
yarn test

# Run a single test file
yarn test path/to/file.test.tsx

# Lint
yarn eslint src

# Extract i18n strings from source code
yarn i18n:extract
yarn i18n:extract --module=module-name

# Update translations from Zendesk translation system
yarn i18n:update-translations
yarn i18n:update-translations --module=module-name
```

**Note:** Preview requires Zendesk authentication. Run `yarn zcli login -i` first if needed.

## Architecture

### Build System
- **Rollup** compiles everything - outputs `script.js`, `style.css`, and ES module bundles in `assets/`
- **Do not edit** `script.js`, `style.css`, or files in `assets/` directly - they are generated
- ES2015 only for `script.js` (no Babel) - avoid newer JavaScript features in `src/*.js` files

### Directory Structure
- `templates/` - Curlybars templates (`.hbs`) for Help Center pages. Curlybars is a subset of Handlebars and may not support all Handlebars features.
- `src/` - JavaScript source
  - `src/*.js` - Legacy vanilla JS (bundled into `script.js` as IIFE)
  - `src/modules/` - React components (bundled as ES modules into `assets/`)
- `styles/` - SCSS files (compiled into `style.css`)
- `manifest.json` - Theme settings configuration

### React Modules
Located in `src/modules/`, bundled to `assets/*-bundle.js`:
- `new-request-form` - Ticket submission form
- `request-list` - User's requests page
- `service-catalog` - Service catalog pages
- `approval-requests` - Approval workflow UI
- `flash-notifications` - Toast notifications
- `shared/` - Shared utilities (Garden theme, i18n, error boundary)
- `ticket-fields/` - Reusable ticket field components

Modules are isolated - ESLint enforces that modules only import from `shared/`, `test/`, `ticket-fields/`, or `flash-notifications/`.

### Import Maps
React modules are loaded via import maps generated during build. The `document_head.hbs` template contains the import map mapping module names to asset URLs.

### i18n in React
Uses react-i18next with `.` separator for plurals (not `_`). Translation strings use:
```ts
const { t } = useTranslation();
t("key", "Default English value");
```
Translations stored in `src/modules/[module]/translations/`.

### Translations: what goes where
- `translations.yml` (repo root) is only for strings referenced by `manifest.json` (theme settings labels/descriptions shown in the Settings panel). Don't add template or React strings here.
- In Curlybars templates (`templates/*.hbs`), the `{{t "key"}}` helper only resolves keys that Help Center exposes to the theme's `t` helper. A new key must be added there first; defining it in `translations.yml` or a module's `translations/` folder does not make it available to `{{t}}`.
- React component strings live in `src/modules/[module]/translations/` and are loaded via `react-i18next` (see README for the extraction/update workflow).

## File Naming Conventions

- Folders in `src/`: kebab-case
- TypeScript/JavaScript files in `src/`: PascalCase or camelCase

## APIs

Use only public REST APIs related to the help center documented at:

- [Introduction](/api-reference/help_center/help-center-templates/introduction/)
- [Helpers](https://developer.zendesk.com/api-reference/help_center/help-center-templates/helpers/)
- [Advanced helpers](/api-reference/help_center/help-center-templates/advanced_helpers/)
- [Objects](/api-reference/help_center/help-center-templates/objects/)
- [Header](/api-reference/help_center/help-center-templates/header_page/)
- [Footer](/api-reference/help_center/help-center-templates/footer_page/)
- [Home page](/api-reference/help_center/help-center-templates/home_page/)
- [Category page](/api-reference/help_center/help-center-templates/category_page/)
- [Section page](/api-reference/help_center/help-center-templates/section_page/)
- [Article page](/api-reference/help_center/help-center-templates/article_page/)
- [Contributions page](/api-reference/help_center/help-center-templates/contributions_page/)
- [Following page](/api-reference/help_center/help-center-templates/following_page/)
- [Request List page](/api-reference/help_center/help-center-templates/request_list_page/)
- [Request page](/api-reference/help_center/help-center-templates/request_page/)
- [New Request page](/api-reference/help_center/help-center-templates/new_request_page/)
- [Search Results page](/api-reference/help_center/help-center-templates/search_results_page/)
- [Service catalog page templates](/api-reference/help_center/help-center-templates/service_catalog/)
- [Approval request page templates](/api-reference/help_center/help-center-templates/approval_requests/)
- [Error page](/api-reference/help_center/help-center-templates/error_page/)
- [New Community Post page](/api-reference/help_center/help-center-templates/new_community_post_page/)
- [Community Post page](/api-reference/help_center/help-center-templates/community_post_page/)
- [Community Post List page](/api-reference/help_center/help-center-templates/community_post_list_page/)
- [Community Topic page](/api-reference/help_center/help-center-templates/community_topic_page/)
- [Community Topic List page](/api-reference/help_center/help-center-templates/community_topic_list_page/)
- [User Profile page](/api-reference/help_center/help-center-templates/user_profile_page/)
- [Custom page](/api-reference/help_center/help-center-templates/custom_page/)

Always attempt to use context7 to review the Zendesk Developer docs for the above, but you can use the above links as a fallback if needed. These are the only developer docs related to Help Center theming.

## Previewing

You can use ZCLI to preview the theme build using Playwright (only Firefox will work due to a CORS bug/issue currently when using a Chromium browser - do not use chrome to preview)

```
zcli themes:preview

Uploading theme... Ok
Ready https://glean-73565.zendesk.com/hc/admin/local_preview/start 🚀
You can exit preview mode in the UI or by visiting https://glean-73565.zendesk.com/hc/admin/local_preview/stop
```

You will be asked to login with a user. You will need to pause and ask the user to login using their Zendesk credentials + MFA for you to be able to proceed.

## Coverly theme conventions

This is a Coverly-branded Help Center theme. A few conventions are worth knowing when authoring or extending:

### Category icons

The home page renders a category icon next to each `.cat-card`. Icons are inline SVG, keyed off the category **name** (not id, because Zendesk's `category.id` is numeric), in `templates/home_page.hbs`. The current mapping covers:

- `"Getting Started with Coverly"` → compass
- `"Buying Plans & Billing"` → wallet
- `"Installing & Activating Your eSIM"` → download
- `"Device Compatibility"` → smartphone
- `"Coverage & Networks"` → globe
- `"Using the Coverly Mobile App"` → smartphone
- `"Account, Privacy & Security"` → shield
- `"Troubleshooting"` → wrench
- `"Destination & Region Guides"` → suitcase
- `"Coverly for Business"` → briefcase
- `"Coverly Stores & Airport Kiosks"` → map-pin
- `"About Coverly"` → info
- *any other name* → generic document icon (fallback)

To add a new icon, edit the `{{#is name "…"}}` chain in `home_page.hbs` and add a new `{{else}}{{#is name "Your Display Name"}}…<svg>…{{/is}}` branch before the fallback. The chain must end with one `{{/is}}` per `{{#is}}` opener (currently 12 of each).

### Article body styling

Standard WYSIWYG output (h2, h3, h4, p, ul, ol, blockquote, code, pre, img, table) is styled by default. Two opt-in classes provide bespoke editorial treatments:

- **`.callout`** — accent-bordered block for tips and important notes. Wrap your callout text in `<div class="callout">…</div>` in the article HTML source.
- **`.step-card`** — numbered card for step-by-step instructions. Markup:
  ```html
  <div class="step-card">
    <div class="step-num">01</div>
    <div class="step-body">
      <strong>Headline</strong>
      <span>Detail copy</span>
    </div>
  </div>
  ```

Authors who want these treatments need to paste the raw HTML via the WYSIWYG editor's source-edit mode.

### Branded assets

- `assets/coverly-icon.png` — default logo, overridden when `settings.logo` is set
- `assets/umbrellas.jpg` — default hero background, referenced via the Zass-injected SCSS variable `$assets-umbrellas-jpg` in `styles/_hero.scss`

The Zass plugin (`zass.mjs`) automatically generates `$assets-<name>-<ext>` SCSS variables for every file in `assets/`. Reference them in SCSS as `url($assets-name-ext)` — the literal token is preserved through compilation and Zendesk substitutes the real CDN URL server-side at request time.

### Dark theme is hardcoded

`manifest.json` no longer exposes color or font settings. The palette and typography are baked into `styles/_variables.scss` (SCSS) and `styles/_base.scss` (CSS custom properties on `:root`). Adjust both files together when changing tokens.

### Article comments dropped

The article page template (`templates/article_page.hbs`) does not render comments. If you need comments back, restore the `{{#if settings.show_article_comments}}` block from Copenhagen's original article_page.hbs and re-add the setting to `manifest.json`.

### Out-of-scope pages

This migration explicitly only restyles home, category, section, and article pages plus header/footer/document_head. Other pages (search results, request forms, community, profile, service catalog, approval requests, error pages, contributions) use Copenhagen's existing templates but inherit the new dark palette via SCSS variable rebinding in `_variables.scss`. They may have minor styling quirks since they weren't designed for a dark background.