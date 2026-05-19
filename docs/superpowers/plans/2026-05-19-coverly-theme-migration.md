# Coverly Theme Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate the Coverly Support Center design (Claude Design handoff bundle in `coverly-support-center/`) into this Copenhagen-based Zendesk Help Center theme. Restyle home, category, section, and article pages plus shared header/footer to pixel-match the bundle. Out-of-scope pages inherit the new palette but keep Copenhagen markup.

**Architecture:** Convert Coverly's `styles.css` into SCSS partials that replace the existing Copenhagen partials at the same paths. Tokens are exposed as CSS custom properties on `:root` (consumed directly by new partials) and as SCSS variables (consumed by untouched legacy partials via Zass injection). Rewrite the four focus templates plus header / footer / document_head; leave every other template untouched. React modules in `src/modules/` are unchanged and inherit colors via CSS variables.

**Tech Stack:** Curlybars (Handlebars subset) templates, SCSS via Rollup + custom Zass plugin (`zass.mjs`), Google Fonts (Newsreader, Manrope, JetBrains Mono), packaged image assets via `{{asset}}` helper. Local preview through `zcli themes:preview` (Firefox only — Chromium has a CORS bug).

**Reference**: All Coverly source is in `coverly-support-center/project/`:
- `styles.css` — design CSS (933 lines, well-organized by component)
- `components.jsx` — Header / Footer / Icon SVG dictionary / ArticleRow
- `pages-main.jsx` — Home / Category / Section / Article page markup

**Spec**: `docs/superpowers/specs/2026-05-19-coverly-theme-design.md` (read this if any task is ambiguous).

---

## Phase 1: Foundation

These five tasks change tokens and load fonts without altering page markup. After Phase 1, every existing Copenhagen template should still render but with the new dark palette and Newsreader/Manrope typography. Visual ugly is expected mid-phase; verification is `yarn build` success.

### Task 1: Ignore the design bundle

The bundle in `coverly-support-center/` is a reference artifact, not part of the shipped theme. Keep it on disk for the implementer but don't ship it.

**Files:**
- Modify: `.gitignore`

- [ ] **Step 1: Check current `.gitignore`**

Run: `cat .gitignore`

- [ ] **Step 2: Append the bundle path**

Append this block to `.gitignore` (preserve existing entries):

```
# Claude Design handoff bundle — reference only, not part of shipped theme
coverly-support-center/
```

- [ ] **Step 3: Verify git no longer tracks it**

Run: `git status --short`
Expected: `coverly-support-center/` is no longer in untracked output.

- [ ] **Step 4: Commit**

```bash
git add .gitignore
git commit -m "chore: ignore coverly-support-center handoff bundle"
```

---

### Task 2: Add packaged image assets

Ship Coverly's logo and hero photo as theme assets so we can reference them via `{{asset}}` instead of admin-uploadable settings.

**Files:**
- Create: `assets/coverly-icon.png`
- Create: `assets/umbrellas.jpg`

- [ ] **Step 1: Copy logo**

Run: `cp coverly-support-center/project/assets/coverly-icon.png assets/coverly-icon.png`

- [ ] **Step 2: Copy hero photo**

Run: `cp coverly-support-center/project/assets/umbrellas.jpg assets/umbrellas.jpg`

- [ ] **Step 3: Verify both files exist**

Run: `ls -la assets/coverly-icon.png assets/umbrellas.jpg`
Expected: both files present, non-zero size.

- [ ] **Step 4: Commit**

```bash
git add assets/coverly-icon.png assets/umbrellas.jpg
git commit -m "feat(theme): add Coverly logo and hero photo assets"
```

---

### Task 3: Wire Google Fonts in document_head

Load Newsreader, Manrope, and JetBrains Mono so the new SCSS can use them.

**Files:**
- Modify: `templates/document_head.hbs`

- [ ] **Step 1: Read current document_head**

Read `templates/document_head.hbs` to understand existing import-map and module bootstrap (do not touch those — they're required by React modules).

- [ ] **Step 2: Add font link tags**

Replace the entire file with:

```hbs
<meta content="width=device-width, initial-scale=1.0" name="viewport" />

<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Newsreader:ital,opsz,wght@0,6..72,400;0,6..72,500;0,6..72,600;1,6..72,400&family=Manrope:wght@400;500;600;700&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet">

<!-- Make the translated search clear button label available for use in JS -->
<!-- See buildClearSearchButton() in script.js -->
<script type="text/javascript">window.searchClearButtonLabelLocalized = "{{t 'search_clear'}}";</script>
<script type="text/javascript">
  // Load ES module polyfill only for browsers that don't support ES modules
  if (!(HTMLScriptElement.supports && HTMLScriptElement.supports('importmap'))) {
    document.write('<script async src="{{asset 'es-module-shims.js'}}"><\/script>');
  }
</script>
<script type="importmap">
{
  "imports": {
    "new-request-form": "{{asset 'new-request-form-bundle.js'}}",
    "request-list": "{{asset 'request-list-bundle.js'}}",
    "flash-notifications": "{{asset 'flash-notifications-bundle.js'}}",
    "service-catalog": "{{asset 'service-catalog-bundle.js'}}",
    "approval-requests": "{{asset 'approval-requests-bundle.js'}}",
    "approval-requests-translations": "{{asset 'approval-requests-translations-bundle.js'}}",
    "new-request-form-translations": "{{asset 'new-request-form-translations-bundle.js'}}",
    "request-list-translations": "{{asset 'request-list-translations-bundle.js'}}",
    "service-catalog-translations": "{{asset 'service-catalog-translations-bundle.js'}}",
    "shared": "{{asset 'shared-bundle.js'}}",
    "ticket-fields": "{{asset 'ticket-fields-bundle.js'}}",
    "wysiwyg": "{{asset 'wysiwyg-bundle.js'}}"
  }
}
</script>
<script type="module">
  import { renderFlashNotifications } from "flash-notifications";

  const settings = {{json settings}};
  const baseLocale = {{json help_center.base_locale}};

  renderFlashNotifications(settings, baseLocale);
</script>
```

- [ ] **Step 3: Verify build succeeds**

Run: `yarn build`
Expected: build completes without errors. Note that fonts won't be visually loaded yet since CSS hasn't been changed — that's fine.

- [ ] **Step 4: Commit**

```bash
git add templates/document_head.hbs
git commit -m "feat(head): load Newsreader, Manrope, JetBrains Mono fonts"
```

---

### Task 4: Rewrite SCSS variables

Rebind the SCSS variables that Copenhagen partials consume so untouched partials (forms, comments, pagination, etc.) automatically inherit the Coverly palette.

**Files:**
- Modify: `styles/_variables.scss`

- [ ] **Step 1: Replace file contents**

Replace the entire contents of `styles/_variables.scss` with:

```scss
// Variables — rebound to the Coverly palette so legacy Copenhagen partials
// inherit the new design without further changes. The new partials reference
// CSS custom properties directly (defined in :root in _base.scss).

// Coverly palette (mirrors the --tokens defined in _base.scss :root)
$coverly-bg:           #0C172F;
$coverly-bg-elev:      #142446;
$coverly-bg-tint:      #1B2C53;
$coverly-ink:          #F2F4F9;
$coverly-ink-soft:     #B6C3DD;
$coverly-ink-mute:     #6E7FA4;
$coverly-line:         #25366A;
$coverly-line-strong:  #38498A;
$coverly-primary:      #5BE0B0;
$coverly-primary-ink:  #0B2A22;
$coverly-accent:       #6FA3E8;
$coverly-mint-deep:    #2BB089;

// Legacy Zass-injected vars (originally driven by manifest.json settings,
// now hardcoded since manifest no longer exposes color/font settings).
// Names must match what existing partials reference.
$brand_color:          $coverly-primary;
$brand_text_color:     $coverly-primary-ink;
$text_color:           $coverly-ink;
$background_color:     $coverly-bg;
$link_color:           $coverly-primary;
$hover_link_color:     $coverly-mint-deep;
$visited_link_color:   $coverly-accent;
$heading_font:         "Newsreader", "Georgia", serif;
$text_font:            "Manrope", system-ui, sans-serif;

// Derived
$hover-button-color:   $coverly-mint-deep;
$secondary-text-color: $coverly-ink-soft;
$primary-shade:        $coverly-bg-elev;
$secondary-shade:      $coverly-bg-tint;

$font-size-bigger:     16px;
$font-size-base:       15px;
$font-size-navigation: 14px;
$font-size-small:      13px;
$font-size-smaller:    11px;

$high-contrast-border-color: $coverly-line-strong;
$low-contrast-border-color:  $coverly-line;
$menu-border-color:          $coverly-line;

$button-color:               $coverly-primary;

$input-font-size:            14px;
$field-text-color:           $coverly-ink-soft;
$field-text-focus-color:     $coverly-ink;

// Breakpoints
$tablet-width:         768px;
$desktop-width:        1024px;
$max-width-container:  1200px; // bumped from 1160px to match Coverly's .wrap

$font-weight-semibold: 600;

$input-transition:     border .12s ease-in-out;
```

- [ ] **Step 2: Verify build succeeds**

Run: `yarn build`
Expected: build completes. Some legacy partials may produce SCSS warnings about color contrast on the new dark background — these are not errors and are acceptable per the spec.

- [ ] **Step 3: Commit**

```bash
git add styles/_variables.scss
git commit -m "feat(styles): rebind SCSS tokens to Coverly palette"
```

---

### Task 5: Rewrite SCSS base + tokens

Define the full CSS custom property token set on `:root` (consumed by new partials), set body typography/colors, and define the small set of utility classes (`.display`, `.mono`, `.wrap`, `.tag`, `.link-u`, `.status-chip`, `.status-dot`) used across multiple pages.

**Files:**
- Modify: `styles/_base.scss`

- [ ] **Step 1: Replace file contents**

Replace the entire contents of `styles/_base.scss` with:

```scss
/***** Coverly tokens — defined as CSS custom properties on :root so new
       partials reference --token, and React modules inherit via inheritance. *****/

:root {
  // Brand
  --brand-navy:      #1F3A6E;
  --brand-cobalt:    #2E66C7;
  --brand-sky:       #6FA3E8;
  --brand-mint:      #5BE0B0;
  --brand-mint-deep: #2BB089;

  // Surface
  --bg:           #0C172F;
  --bg-elev:      #142446;
  --bg-tint:      #1B2C53;
  --ink:          #F2F4F9;
  --ink-soft:     #B6C3DD;
  --ink-mute:     #6E7FA4;
  --line:         #25366A;
  --line-strong:  #38498A;

  // Action
  --primary:      #5BE0B0;
  --primary-ink:  #0B2A22;
  --accent:       #6FA3E8;
  --accent-ink:   #0C172F;

  // Hero overlay & shadow
  --hero-overlay: linear-gradient(180deg, rgba(8, 20, 45, 0.6) 0%, rgba(8, 20, 45, 0.85) 100%);
  --shadow-card:  0 1px 0 rgba(0, 0, 0, 0.4), 0 12px 32px -16px rgba(0, 0, 0, 0.6);

  // Type
  --font-display:     "Newsreader", "Georgia", serif;
  --font-body:        "Manrope", system-ui, sans-serif;
  --font-mono:        "JetBrains Mono", ui-monospace, monospace;
  --display-weight:   500;
  --display-tracking: -0.02em;
}

/***** Base reset *****/
* { box-sizing: border-box; }

html, body { margin: 0; padding: 0; }

body {
  background: var(--bg);
  color: var(--ink);
  font-family: var(--font-body);
  font-size: 16px;
  line-height: 1.55;
  -webkit-font-smoothing: antialiased;
  text-rendering: optimizeLegibility;
  min-height: 100vh;

  > main {
    @include desktop {
      min-height: 65vh;
    }
  }
}

a {
  color: inherit;
  text-decoration: none;

  &:hover,
  &:active,
  &:focus {
    text-decoration: none;
  }
}

button { font-family: inherit; cursor: pointer; }
img { max-width: 100%; display: block; }

ul {
  list-style: none;
  margin: 0;
  padding: 0;
}

/***** Form input baseline (consumed by hbs-form / .search and legacy partials) *****/
.hbs-form, .search {
  input,
  textarea {
    color: var(--ink);
    font-size: $input-font-size;
    background: var(--bg-elev);
  }

  input {
    max-width: 100%;
    box-sizing: border-box;
    transition: $input-transition;

    &:where(:not([type="checkbox"])) {
      outline: none;
      border: 1px solid var(--line-strong);

      &:focus {
        border: 1px solid var(--primary);
      }
    }
  }

  input[disabled] {
    background-color: var(--bg-tint);
  }

  select {
    -webkit-appearance: none;
    -moz-appearance: none;
    background: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='6' viewBox='0 0 10 6'%3E%3Cpath fill='%236E7FA4' d='M0 0h10L5 6 0 0z'/%3E%3C/svg%3E") no-repeat var(--bg-elev);
    background-position: right 10px center;
    border: 1px solid var(--line-strong);
    border-radius: 4px;
    padding: 8px 30px 8px 10px;
    outline: none;
    color: var(--ink);
    width: 100%;

    &:focus {
      border: 1px solid var(--primary);
    }

    &::-ms-expand { display: none; }
  }

  textarea {
    border: 1px solid var(--line-strong);
    border-radius: 6px;
    resize: vertical;
    width: 100%;
    outline: none;
    padding: 10px;

    &:focus {
      border: 1px solid var(--primary);
    }
  }
}

/***** Typography utilities *****/
.display {
  font-family: var(--font-display);
  font-weight: var(--display-weight);
  letter-spacing: var(--display-tracking);
  line-height: 1.05;
}

.mono {
  font-family: var(--font-mono);
  font-size: 0.78rem;
  letter-spacing: 0.06em;
  text-transform: uppercase;
}

.muted { color: var(--ink-mute); }
.soft  { color: var(--ink-soft); }

/***** Layout wrappers *****/
.wrap        { max-width: 1200px; margin: 0 auto; padding: 0 32px; }
.wrap-narrow { max-width: 820px;  margin: 0 auto; padding: 0 32px; }

// Legacy container — preserve for untouched templates.
.container {
  @include max-width-container;
}

.container-divider {
  border-top: 1px solid var(--line);
  margin-bottom: 20px;
}

.error-page {
  @include max-width-container;
}

.visibility-hidden {
  @include visually-hidden;
}

/***** Tags *****/
.tag {
  display: inline-flex;
  align-items: center;
  padding: 3px 10px;
  background: var(--bg-tint);
  color: var(--ink-soft);
  border-radius: 999px;
  font-size: 0.76rem;
  font-weight: 500;
  border: 1px solid transparent;
  transition: all .15s;

  &:hover  { border-color: var(--line-strong); }
  &.active { background: var(--primary); color: var(--primary-ink); border-color: var(--primary); }
  &.outline { background: transparent; border-color: var(--line); }
}

/***** Underline link *****/
.link-u {
  color: var(--primary);
  font-weight: 600;
  border-bottom: 1px solid color-mix(in oklab, var(--primary) 35%, transparent);
  display: inline-flex;
  align-items: center;
  gap: 4px;

  &:hover { border-bottom-color: var(--primary); }
}

/***** Status chip & dot *****/
.status-chip {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  font-size: 0.82rem;
  color: var(--ink-soft);
}

.status-dot {
  width: 8px;
  height: 8px;
  border-radius: 999px;
  background: var(--brand-mint-deep);
  box-shadow: 0 0 0 4px color-mix(in oklab, var(--brand-mint) 30%, transparent);
  animation: pulse 2.2s ease-out infinite;
}

@keyframes pulse {
  0%   { box-shadow: 0 0 0 0 color-mix(in oklab, var(--brand-mint) 50%, transparent); }
  70%  { box-shadow: 0 0 0 8px color-mix(in oklab, var(--brand-mint) 0%,  transparent); }
  100% { box-shadow: 0 0 0 0 color-mix(in oklab, var(--brand-mint) 0%,  transparent); }
}
```

- [ ] **Step 2: Verify build succeeds**

Run: `yarn build`
Expected: clean build. `style.css` is regenerated.

- [ ] **Step 3: Sanity-check rendered output**

Run: `grep -c "var(--primary)" style.css`
Expected: a positive integer (custom properties are present in the compiled CSS).

- [ ] **Step 4: Commit**

```bash
git add styles/_base.scss
git commit -m "feat(styles): define Coverly :root tokens and base utilities"
```

---

## Phase 2: Shell — header & footer

Replaces the chrome that appears on every page. After Phase 2, every page in the theme (including out-of-scope ones) renders with the new Coverly header and footer.

### Task 6: Rewrite `_header.scss`

**Files:**
- Modify: `styles/_header.scss`

- [ ] **Step 1: Reference**

Read lines 138–184 of `coverly-support-center/project/styles.css` (covers `.header`, `.header-inner`, `.brand`, `.brand-name`, `.brand-sub`, `.nav`, `.nav .pill`, `.status-chip`, `.status-dot`, `@keyframes pulse`).

- [ ] **Step 2: Replace file contents**

Replace the entire contents of `styles/_header.scss` with:

```scss
.header {
  position: sticky;
  top: 0;
  z-index: 40;
  background: color-mix(in oklab, var(--bg) 92%, transparent);
  backdrop-filter: blur(14px);
  border-bottom: 1px solid var(--line);
}

.header-inner {
  display: flex;
  align-items: center;
  gap: 32px;
  padding: 18px 32px;
  max-width: 1280px;
  margin: 0 auto;
}

.brand {
  display: flex;
  align-items: center;
  gap: 12px;

  img {
    width: 32px;
    height: 32px;
  }
}

.brand-name {
  font-family: var(--font-display);
  font-weight: var(--display-weight);
  font-size: 1.6rem;
  letter-spacing: var(--display-tracking);
  color: var(--ink);
}

.brand-sub {
  font-size: 0.78rem;
  color: var(--ink-mute);
  margin-left: 4px;
}

.nav {
  display: flex;
  gap: 28px;
  margin-left: auto;
  align-items: center;

  a {
    font-size: 0.92rem;
    color: var(--ink-soft);
    transition: color .15s;

    &:hover { color: var(--ink); }
  }

  .pill {
    background: var(--primary);
    color: var(--primary-ink);
    padding: 9px 18px;
    border-radius: 999px;
    font-weight: 600;
    font-size: 0.88rem;

    &:hover {
      color: var(--primary-ink);
      opacity: 0.92;
    }
  }
}

/***** Mobile drawer (adapted from Copenhagen, restyled for dark surface) *****/
.nav-wrapper-mobile {
  display: none;

  @media (max-width: 800px) {
    display: block;
    margin-left: auto;
  }
}

.nav-wrapper-desktop {
  @media (max-width: 800px) {
    display: none;
  }
}

.menu-button-mobile {
  background: transparent;
  border: 1px solid var(--line);
  border-radius: 8px;
  color: var(--ink);
  padding: 8px 10px;
  display: inline-flex;
  align-items: center;
  gap: 8px;

  .icon-menu { color: var(--ink); }
}

.menu-list-mobile {
  display: none;
  background: var(--bg-elev);
  border: 1px solid var(--line);
  border-radius: 10px;
  padding: 14px;
  position: absolute;
  right: 32px;
  top: 64px;
  min-width: 240px;
  box-shadow: var(--shadow-card);

  &[aria-expanded="true"] { display: block; }

  ul { list-style: none; padding: 0; margin: 0; }
  li.item { padding: 8px 4px; }
  li.item a { color: var(--ink-soft); font-size: 0.92rem; }
  li.item a:hover { color: var(--ink); }
  .nav-divider {
    border-top: 1px solid var(--line);
    margin: 10px 0;
  }
}

.skip-navigation {
  position: absolute;
  left: -10000px;

  &:focus {
    left: 16px;
    top: 16px;
    background: var(--bg-elev);
    color: var(--ink);
    padding: 8px 12px;
    border-radius: 6px;
    z-index: 100;
  }
}

/***** Signed-in user dropdown (uses Copenhagen .dropdown markup) *****/
.user-info {
  margin-left: 8px;

  .dropdown-toggle {
    background: transparent;
    border: 1px solid var(--line);
    color: var(--ink);
    padding: 6px 12px;
    border-radius: 999px;
    display: inline-flex;
    align-items: center;
    gap: 8px;

    .user-avatar {
      width: 24px;
      height: 24px;
      border-radius: 999px;
    }
  }
}
```

- [ ] **Step 3: Verify build succeeds**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 4: Commit**

```bash
git add styles/_header.scss
git commit -m "feat(styles): port Coverly header styles"
```

---

### Task 7: Rewrite `templates/header.hbs`

Replace Copenhagen header markup with Coverly's structure, wired to Zendesk helpers. Drop the "Ask AI" link entirely.

**Files:**
- Modify: `templates/header.hbs`

- [ ] **Step 1: Reference**

Read `coverly-support-center/project/components.jsx` lines 66–96 (the `<Header>` component).

- [ ] **Step 2: Replace file contents**

Replace the entire contents of `templates/header.hbs` with:

```hbs
<a class="skip-navigation" tabindex="1" href="#main-content">{{t 'skip_navigation'}}</a>

<header class="header">
  <div class="header-inner">
    {{#link 'help_center' class='brand'}}
      {{#if settings.logo}}
        <img src="{{settings.logo}}" alt="{{t 'home_page' name=help_center.name}}" />
      {{else}}
        <img src="{{asset 'coverly-icon.png'}}" alt="{{t 'home_page' name=help_center.name}}" />
      {{/if}}
      <span>
        {{#if settings.show_brand_name}}
          <span class="brand-name">{{help_center.name}}</span>
        {{/if}}
        <span class="brand-sub"> / Help Center</span>
      </span>
    {{/link}}

    <div class="nav-wrapper-desktop">
      <nav class="nav" id="user-nav" aria-label="{{t 'user_navigation'}}">
        {{link 'community'}}
        {{link 'service_catalog'}}

        <span class="status-chip" title="All systems operational">
          <span class="status-dot"></span> All systems normal
        </span>

        {{#link 'new_request' class='pill submit-a-request'}}
          {{t 'submit_a_request'}}
        {{/link}}

        {{#unless signed_in}}
          {{#link "sign_in" class="sign-in" role="link"}}
            {{t 'sign_in'}}
          {{/link}}
        {{/unless}}

        {{#if signed_in}}
          <div class="user-info dropdown">
            <button class="dropdown-toggle" aria-haspopup="true" aria-expanded="false">
              {{user_avatar class="user-avatar"}}
              <span>
                {{user_name}}
                <svg xmlns="http://www.w3.org/2000/svg" width="12" height="12" focusable="false" viewBox="0 0 12 12" class="dropdown-chevron-icon" aria-hidden="true">
                  <path fill="none" stroke="currentColor" stroke-linecap="round" d="M3 4.5l2.6 2.6c.2.2.5.2.7 0L9 4.5"/>
                </svg>
              </span>
            </button>
            <div class="dropdown-menu" role="menu">
              {{#my_profile role="menuitem"}}{{t 'profile'}}{{/my_profile}}
              {{link "requests" role="menuitem"}}
              {{#link "contributions" role="menuitem"}}{{t "activities"}}{{/link}}
              {{link "approval_requests" role="menuitem"}}
              {{contact_details role="menuitem"}}
              {{change_password role="menuitem"}}
              <div class="separator" role="separator"></div>
              {{link "sign_out" role="menuitem"}}
            </div>
          </div>
        {{/if}}
      </nav>
    </div>

    <div class="nav-wrapper-mobile">
      <button class="menu-button-mobile" aria-controls="user-nav-mobile" aria-expanded="false" aria-label="{{t 'toggle_navigation'}}">
        {{#if signed_in}}{{user_avatar class="user-avatar"}}{{/if}}
        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" focusable="false" viewBox="0 0 16 16" class="icon-menu">
          <path fill="none" stroke="currentColor" stroke-linecap="round" d="M1.5 3.5h13m-13 4h13m-13 4h13"/>
        </svg>
      </button>
      <nav class="menu-list-mobile" id="user-nav-mobile" aria-expanded="false">
        <ul class="menu-list-mobile-items">
          {{#if signed_in}}
            <li class="user-avatar-item item">
              {{#my_profile class="my-profile"}}
                {{user_avatar class="menu-profile-avatar"}}
                <div class="menu-profile-name">
                  <div>{{user_name}}</div>
                  <div class="my-profile-tooltip">{{t 'profile'}}</div>
                </div>
              {{/my_profile}}
            </li>
            <li class="item">{{link "requests"}}</li>
            <li class="item">{{#link "contributions"}}{{t "activities"}}{{/link}}</li>
            <li class="item">{{link "approval_requests"}}</li>
            <li class="item">{{contact_details}}</li>
            <li class="item">{{change_password class='change-password'}}</li>
            <li class="nav-divider"></li>
          {{else}}
            <li class="item">{{#link "sign_in" role="link"}}{{t 'sign_in'}}{{/link}}</li>
            <li class="nav-divider"></li>
          {{/if}}

          <li class="item">{{link 'community'}}</li>
          <li class="item">{{link 'new_request' class='submit-a-request'}}</li>
          <li class="item">{{link 'service_catalog'}}</li>
          <li class="nav-divider"></li>

          {{#if signed_in}}
            <li class="item">{{link "sign_out"}}</li>
          {{/if}}
        </ul>
      </nav>
    </div>
  </div>
</header>
```

- [ ] **Step 3: Verify build succeeds**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 4: Commit**

```bash
git add templates/header.hbs
git commit -m "feat(header): rewrite to Coverly markup, drop Ask AI link"
```

---

### Task 8: Rewrite `_footer.scss`

**Files:**
- Modify: `styles/_footer.scss`

- [ ] **Step 1: Reference**

Read lines 876–897 of `coverly-support-center/project/styles.css` (covers `.footer`, `.footer-grid`, `.footer h5`, `.footer ul`, `.footer .signature`, `.footer-bottom`).

- [ ] **Step 2: Replace file contents**

Replace the entire contents of `styles/_footer.scss` with:

```scss
.footer {
  border-top: 1px solid var(--line);
  background: var(--bg-tint);
  padding: 56px 0 32px;
  margin-top: 80px;
}

.footer-inner {
  // Keep selector for untouched templates that still emit it.
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 32px;
}

.footer-grid {
  display: grid;
  grid-template-columns: 1.4fr repeat(3, 1fr);
  gap: 40px;

  @media (max-width: 700px) {
    grid-template-columns: 1fr 1fr;
  }
}

.footer {
  h5 {
    font-size: 0.78rem;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: var(--ink-mute);
    margin: 0 0 14px;
  }

  ul {
    list-style: none;
    padding: 0;
    margin: 0;
    display: flex;
    flex-direction: column;
    gap: 8px;

    a {
      font-size: 0.92rem;
      color: var(--ink-soft);

      &:hover { color: var(--ink); }
    }
  }

  .signature {
    font-size: 0.92rem;
    color: var(--ink-soft);
    max-width: 36ch;
    margin-top: 12px;
  }

  .brand img { width: 32px; height: 32px; }
  .brand .brand-name { font-size: 1.4rem; }
}

.footer-bottom {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-top: 1px solid var(--line);
  padding-top: 22px;
  margin-top: 42px;
  font-size: 0.84rem;
  color: var(--ink-mute);

  @media (max-width: 700px) {
    flex-direction: column;
    gap: 14px;
    align-items: flex-start;
  }
}

.footer-language-selector {
  // Inline language selector — uses Copenhagen .dropdown markup.
  .dropdown-toggle {
    background: transparent;
    border: 1px solid var(--line);
    color: var(--ink-soft);
    padding: 6px 14px;
    border-radius: 999px;
    font-size: 0.84rem;
    display: inline-flex;
    align-items: center;
    gap: 6px;

    &:hover { color: var(--ink); }
  }
}
```

- [ ] **Step 3: Verify build succeeds**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 4: Commit**

```bash
git add styles/_footer.scss
git commit -m "feat(styles): port Coverly footer styles"
```

---

### Task 9: Rewrite `templates/footer.hbs`

**Files:**
- Modify: `templates/footer.hbs`

- [ ] **Step 1: Reference**

Read `coverly-support-center/project/components.jsx` lines 98–146 (the `<Footer>` component).

- [ ] **Step 2: Replace file contents**

Replace the entire contents of `templates/footer.hbs` with:

```hbs
<footer class="footer">
  <div class="footer-inner">
    <div class="footer-grid">
      <div>
        <div class="brand" style="margin-bottom: 14px;">
          {{#if settings.logo}}
            <img src="{{settings.logo}}" alt="" />
          {{else}}
            <img src="{{asset 'coverly-icon.png'}}" alt="" />
          {{/if}}
          <span class="brand-name">{{help_center.name}}</span>
        </div>
        <p class="signature">
          Travel eSIMs that just work — in 190+ countries, no roaming surprises, no plastic SIM swap.
        </p>
      </div>

      <div>
        <h5>Help</h5>
        <ul>
          <li>{{link 'help_center'}}</li>
          <li>{{link 'new_request'}}</li>
          <li>{{link 'community'}}</li>
          <li>{{link 'service_catalog'}}</li>
        </ul>
      </div>

      <div>
        <h5>Coverly</h5>
        <ul>
          <li><a href="#">About us</a></li>
          <li><a href="#">Press</a></li>
          <li><a href="#">Careers</a></li>
          <li><a href="#">Affiliates</a></li>
        </ul>
      </div>

      <div>
        <h5>Legal</h5>
        <ul>
          <li><a href="#">Terms of service</a></li>
          <li><a href="#">Privacy policy</a></li>
          <li><a href="#">Fair-use policy</a></li>
          <li><a href="#">System status</a></li>
        </ul>
      </div>
    </div>

    <div class="footer-bottom">
      <span>© 2026 Coverly Mobile, Inc. — Made for travelers, by travelers.</span>

      <span style="display: inline-flex; align-items: center; gap: 16px;">
        <span class="status-chip"><span class="status-dot"></span> All systems normal</span>

        <div class="footer-language-selector">
          {{#if alternative_locales}}
            <div class="dropdown language-selector">
              <button class="dropdown-toggle" aria-haspopup="true" aria-expanded="false">
                {{current_locale.name}}
                <svg xmlns="http://www.w3.org/2000/svg" width="12" height="12" focusable="false" viewBox="0 0 12 12" class="dropdown-chevron-icon" aria-hidden="true">
                  <path fill="none" stroke="currentColor" stroke-linecap="round" d="M3 4.5l2.6 2.6c.2.2.5.2.7 0L9 4.5"/>
                </svg>
              </button>
              <span class="dropdown-menu dropdown-menu-end" role="menu">
                {{#each alternative_locales}}
                  <a href="{{url}}" dir="{{direction}}" rel="nofollow" role="menuitem">{{name}}</a>
                {{/each}}
              </span>
            </div>
          {{/if}}
        </div>
      </span>
    </div>
  </div>
</footer>
```

- [ ] **Step 3: Verify build succeeds**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 4: Commit**

```bash
git add templates/footer.hbs
git commit -m "feat(footer): rewrite to Coverly markup with language selector"
```

---

## Phase 3: Home page

After Phase 3, the home page should visually match the Coverly bundle (modulo trending only appearing if promoted articles exist, and the dropped Coverage section). This is the most opinionated set of changes.

### Task 10: Rewrite `_hero.scss`

**Files:**
- Modify: `styles/_hero.scss`

- [ ] **Step 1: Reference**

Read lines 186–367 of `coverly-support-center/project/styles.css` (covers `.hero`, hero variants, `.hero h1`, `.hero p.lead`, `.search-glow`, `.searchbar`, `.quick-chips`, `.hero-stats`).

- [ ] **Step 2: Replace file contents**

Replace the entire contents of `styles/_hero.scss` with:

```scss
.hero {
  position: relative;
  overflow: hidden;
  padding: 96px 0 120px;
  color: #FFFFFF;
  isolation: isolate;
  background: var(--bg-elev);

  &[data-style="photo"] {
    background: var(--bg-elev);

    &::before {
      content: "";
      position: absolute;
      inset: -10px;
      background-image: url("../assets/umbrellas.jpg");
      background-size: cover;
      background-position: center 38%;
      filter: blur(8px) saturate(1.05);
      transform: scale(1.04);
      z-index: -2;
    }

    &::after {
      content: "";
      position: absolute;
      inset: 0;
      background: var(--hero-overlay);
      z-index: -1;
    }
  }

  h1 {
    font-family: var(--font-display);
    font-weight: var(--display-weight);
    letter-spacing: var(--display-tracking);
    line-height: 1.04;
    font-size: clamp(2.4rem, 5.2vw, 4.4rem);
    margin: 22px 0 14px;
    max-width: 22ch;

    em {
      font-style: italic;
      color: var(--brand-mint);
    }
  }

  p.lead {
    font-size: 1.18rem;
    max-width: 52ch;
    opacity: 0.92;
    margin: 0 0 36px;
  }
}

.hero-eyebrow {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  font-size: 0.78rem;
  letter-spacing: 0.14em;
  text-transform: uppercase;
  padding: 7px 14px;
  border: 1px solid rgba(255, 255, 255, 0.35);
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.08);
  backdrop-filter: blur(8px);
}

/***** Animated search glow *****/
.search-glow {
  position: relative;
  border-radius: 16px;
  padding: 2px;
  max-width: 720px;
  isolation: isolate;

  &::before,
  &::after {
    content: "";
    position: absolute;
    inset: -2px;
    border-radius: 18px;
    padding: 3px;
    background: conic-gradient(
      from var(--ang, 0deg),
      transparent 0deg,
      transparent 30deg,
      var(--brand-mint) 50deg,
      rgba(91, 224, 176, 0) 90deg,
      transparent 180deg,
      transparent 210deg,
      var(--brand-sky) 230deg,
      rgba(111, 163, 232, 0) 270deg,
      transparent 360deg
    );
    -webkit-mask:
      linear-gradient(#000 0 0) content-box,
      linear-gradient(#000 0 0);
    -webkit-mask-composite: xor;
            mask-composite: exclude;
    pointer-events: none;
    z-index: 2;
    animation: searchGlowSpin 3.8s linear infinite;
  }

  &::after {
    filter: blur(2px);
    opacity: 0.85;
    animation-delay: -1.9s;
  }

  .searchbar { position: relative; z-index: 1; }
}

@property --ang {
  syntax: '<angle>';
  initial-value: 0deg;
  inherits: false;
}

@keyframes searchGlowSpin {
  to { --ang: 360deg; }
}

@supports not (background: conic-gradient(from 0deg, red, red)) {
  .search-glow::before,
  .search-glow::after { display: none; }
}

.searchbar {
  display: flex;
  align-items: center;
  gap: 0;
  background: var(--bg-elev);
  border-radius: 14px;
  padding: 7px 7px 7px 22px;
  box-shadow: 0 30px 60px -20px rgba(0, 0, 0, 0.4), 0 0 0 1px rgba(255, 255, 255, 0.06);

  svg, .search-icon {
    color: var(--ink-mute);
    flex-shrink: 0;
  }

  input {
    flex: 1;
    border: none;
    outline: none;
    background: transparent;
    font-family: inherit;
    font-size: 1.05rem;
    color: var(--ink);
    padding: 16px 14px;

    &::placeholder { color: var(--ink-mute); }
  }

  button {
    background: var(--primary);
    color: var(--primary-ink);
    border: none;
    border-radius: 9px;
    padding: 14px 22px;
    font-weight: 600;
    font-size: 0.95rem;

    &:hover { opacity: 0.92; }
  }
}

.quick-chips {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  align-items: center;
  margin-top: 22px;

  .chip {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 7px 14px;
    background: rgba(255, 255, 255, 0.12);
    border: 1px solid rgba(255, 255, 255, 0.18);
    backdrop-filter: blur(6px);
    border-radius: 999px;
    font-size: 0.85rem;
    color: #FFFFFF;
    transition: background .15s;

    &:hover { background: rgba(255, 255, 255, 0.2); }
  }
}

.hero-stats {
  display: flex;
  gap: 48px;
  margin-top: 56px;
  flex-wrap: wrap;

  .hero-stat {
    .num {
      font-family: var(--font-display);
      font-size: 2.2rem;
      font-weight: var(--display-weight);
      letter-spacing: var(--display-tracking);
    }

    .lbl {
      font-size: 0.85rem;
      opacity: 0.78;
      margin-top: 4px;
    }
  }
}
```

- [ ] **Step 3: Verify build succeeds**

Run: `yarn build`
Expected: clean build. `umbrellas.jpg` path resolves correctly from `style.css` (which lives at repo root) to `assets/umbrellas.jpg`.

- [ ] **Step 4: Commit**

```bash
git add styles/_hero.scss
git commit -m "feat(styles): port Coverly hero + animated search glow"
```

---

### Task 11: Rewrite `_breadcrumbs.scss`

**Files:**
- Modify: `styles/_breadcrumbs.scss`

- [ ] **Step 1: Replace file contents**

Replace the entire contents of `styles/_breadcrumbs.scss` with:

```scss
// Coverly breadcrumbs — Zendesk's {{breadcrumbs}} helper emits its own markup.
// Style both Coverly's .crumbs class (if used directly) and Zendesk's default.

.crumbs,
.breadcrumbs {
  display: flex;
  gap: 8px;
  align-items: center;
  font-size: 0.88rem;
  color: var(--ink-mute);
  padding: 26px 0;
  flex-wrap: wrap;

  a {
    color: var(--ink-mute);
    transition: color .15s;

    &:hover { color: var(--primary); }
  }

  .sep,
  & > li + li::before {
    color: var(--ink-mute);
    opacity: 0.5;
    content: "/";
    margin-right: 8px;
  }

  .current,
  & > li:last-child {
    color: var(--ink);
  }

  // Zendesk's breadcrumbs helper renders a <ol> with <li> items.
  ol, ul {
    list-style: none;
    padding: 0;
    margin: 0;
    display: flex;
    gap: 8px;
    align-items: center;
    flex-wrap: wrap;
  }
}
```

- [ ] **Step 2: Verify build succeeds**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 3: Commit**

```bash
git add styles/_breadcrumbs.scss
git commit -m "feat(styles): port Coverly breadcrumbs"
```

---

### Task 12: Rewrite `_home-page.scss`

This partial holds everything below the hero on the home page: status banner, categories grid, trending grid, contact CTA, section heads, block spacing.

**Files:**
- Modify: `styles/_home-page.scss`

- [ ] **Step 1: Reference**

Read lines 369–535 of `coverly-support-center/project/styles.css` (covers `.status-banner`, `.section-head`, `.cat-grid`, `.cat-card`, `.article-row`, `.trending-grid`, `.trending-item`, `.block`, `.block-tight`).

Note: `.article-row` rules are reused in category/section pages, so the article-row block is duplicated into `_category.scss` (Task 14) — we keep `.article-row` here too because hero quick-chips, status banner, and other home-only items live in this partial alongside it.

- [ ] **Step 2: Replace file contents**

Replace the entire contents of `styles/_home-page.scss` with:

```scss
/***** Block spacing (used across home + secondary sections) *****/
.block        { padding: 80px 0; }
.block-tight  { padding: 56px 0; }

/***** Section head — eyebrow + display heading + optional see-all *****/
.section-head {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 24px;
  margin: 0 0 28px;

  .eyebrow {
    font-size: 0.76rem;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: var(--ink-mute);
    margin-bottom: 8px;
  }

  h2 {
    font-family: var(--font-display);
    font-weight: var(--display-weight);
    letter-spacing: var(--display-tracking);
    font-size: clamp(1.8rem, 3.2vw, 2.6rem);
    margin: 0;
    line-height: 1.05;
  }

  a.see-all {
    font-size: 0.92rem;
    color: var(--primary);
    font-weight: 600;
    display: inline-flex;
    gap: 6px;

    &:hover { gap: 10px; }
  }
}

/***** Status banner — pulls up over the hero seam *****/
.status-banner {
  background: var(--bg-elev);
  border: 1px solid var(--line);
  border-radius: 14px;
  padding: 18px 22px;
  display: flex;
  align-items: center;
  gap: 16px;
  margin: -56px auto 0;
  max-width: 1200px;
  position: relative;
  z-index: 3;
  box-shadow: var(--shadow-card);

  .dot-wrap {
    width: 36px;
    height: 36px;
    border-radius: 999px;
    display: grid;
    place-items: center;
    background: color-mix(in oklab, var(--brand-mint) 22%, var(--bg-elev));
    flex-shrink: 0;

    .status-dot { width: 10px; height: 10px; }
  }

  h4 {
    margin: 0;
    font-size: 0.98rem;
    font-weight: 600;
    color: var(--ink);
  }

  p {
    margin: 2px 0 0;
    font-size: 0.86rem;
    color: var(--ink-mute);
  }

  a.linkish {
    margin-left: auto;
    font-size: 0.88rem;
    color: var(--primary);
    font-weight: 600;
    display: inline-flex;
    align-items: center;
    gap: 6px;

    &:hover { gap: 10px; }
  }
}

/***** Categories grid *****/
.cat-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 18px;

  @media (max-width: 900px) { grid-template-columns: repeat(2, 1fr); }
  @media (max-width: 580px) { grid-template-columns: 1fr; }
}

.cat-card {
  background: var(--bg-elev);
  border: 1px solid var(--line);
  border-radius: 18px;
  padding: 28px 24px 24px;
  transition: transform .2s ease, border-color .2s ease, box-shadow .2s ease;
  position: relative;
  display: flex;
  flex-direction: column;
  gap: 12px;
  min-height: 200px;
  cursor: pointer;
  color: inherit;

  &:hover {
    transform: translateY(-3px);
    border-color: var(--line-strong);
    box-shadow: var(--shadow-card);
    text-decoration: none;
  }

  .icon {
    width: 44px;
    height: 44px;
    border-radius: 12px;
    display: grid;
    place-items: center;
    background: color-mix(in oklab, var(--primary) 18%, var(--bg-elev));
    color: var(--primary);
    margin-bottom: 4px;
  }

  h3 {
    font-family: var(--font-display);
    font-weight: var(--display-weight);
    letter-spacing: var(--display-tracking);
    font-size: 1.55rem;
    margin: 0;
    color: var(--ink);
  }

  p {
    margin: 0;
    color: var(--ink-soft);
    font-size: 0.92rem;
  }

  .meta {
    margin-top: auto;
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 0.82rem;
    color: var(--ink-mute);
    padding-top: 14px;
    border-top: 1px solid var(--line);

    .arrow {
      color: var(--primary);
      font-weight: 600;
      transition: transform .2s;
    }
  }

  &:hover .meta .arrow { transform: translateX(4px); }
}

/***** Trending grid *****/
.trending-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 0;
  border: 1px solid var(--line);
  border-radius: 18px;
  overflow: hidden;
  background: var(--bg-elev);
  counter-reset: trending;

  @media (max-width: 700px) { grid-template-columns: 1fr; }
}

.trending-item {
  padding: 22px 24px;
  border-bottom: 1px solid var(--line);
  border-right: 1px solid var(--line);
  cursor: pointer;
  display: flex;
  gap: 18px;
  align-items: flex-start;
  transition: background .15s;
  color: inherit;
  counter-increment: trending;

  &:hover { background: var(--bg-tint); text-decoration: none; }
  &:nth-child(2n) { border-right: none; }
  &:nth-last-child(-n+2) { border-bottom: none; }

  @media (max-width: 700px) {
    border-right: none;

    &:nth-last-child(2) { border-bottom: 1px solid var(--line); }
  }

  &::before {
    content: counter(trending, decimal-leading-zero);
    font-family: var(--font-display);
    font-size: 2rem;
    line-height: 1;
    color: var(--accent);
    opacity: 0.9;
    min-width: 36px;
    font-weight: var(--display-weight);
  }

  .item-body {
    flex: 1;
  }

  h4 {
    margin: 0;
    font-size: 1.02rem;
    font-weight: 500;
    color: var(--ink);
  }

  .meta {
    font-size: 0.8rem;
    color: var(--ink-mute);
    margin-top: 4px;
  }
}

/***** Contact CTA block (home page bottom) *****/
.cta-block {
  background: var(--bg-elev);
  border: 1px solid var(--line);
  border-radius: 22px;
  padding: 44px 48px;
  display: grid;
  grid-template-columns: 1.4fr 1fr;
  gap: 32px;
  align-items: center;

  @media (max-width: 800px) {
    grid-template-columns: 1fr;
    padding: 32px 28px;
  }

  .eyebrow {
    font-size: 0.76rem;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: var(--ink-mute);
    margin-bottom: 8px;
  }

  h2.display {
    font-size: 2.2rem;
    margin: 0 0 12px;
  }

  p {
    color: var(--ink-soft);
    margin: 0 0 22px;
    max-width: 48ch;
  }

  .cta-cards {
    display: flex;
    gap: 12px;
    flex-wrap: wrap;
  }
}

.contact-card {
  background: var(--bg-elev);
  border: 1px solid var(--line);
  border-radius: 14px;
  padding: 22px;
  flex: 1;
  min-width: 160px;

  h4 {
    margin: 0 0 6px;
    font-size: 1rem;
    font-weight: 600;
    color: var(--ink);
    display: flex;
    align-items: center;
    gap: 8px;
  }

  p {
    margin: 0;
    font-size: 0.9rem;
    color: var(--ink-soft);
  }
}

/***** Submit button — used in CTA + helpful widget *****/
.submit-btn {
  background: var(--primary);
  color: var(--primary-ink);
  border: none;
  border-radius: 999px;
  padding: 14px 32px;
  font-weight: 600;
  font-size: 0.98rem;
  cursor: pointer;
  transition: opacity .15s;
  display: inline-flex;
  align-items: center;
  gap: 8px;

  &:hover { opacity: 0.92; }
  &:disabled { opacity: 0.5; cursor: not-allowed; }
}
```

- [ ] **Step 3: Verify build succeeds**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 4: Commit**

```bash
git add styles/_home-page.scss
git commit -m "feat(styles): port Coverly home page (status, grid, trending, CTA)"
```

---

### Task 13: Rewrite `templates/home_page.hbs`

Final piece of Phase 3. Wires the home page to Zendesk data using the new markup.

**Files:**
- Modify: `templates/home_page.hbs`

- [ ] **Step 1: Reference**

Read `coverly-support-center/project/pages-main.jsx` lines 7–203 (the `<HomePage>` component).

- [ ] **Step 2: Replace file contents**

Replace the entire contents of `templates/home_page.hbs` with:

```hbs
<h1 class="visibility-hidden">{{help_center.name}}</h1>

<main id="main-content">
  <section class="hero" data-style="photo">
    <div class="wrap">
      <span class="hero-eyebrow">
        <span class="status-dot"></span>
        Coverly Help Center · 190+ countries
      </span>

      <h1 class="display">
        We've got you <em>covered</em> wherever you wander.
      </h1>

      <p class="lead">
        Setup guides, troubleshooting, and travel tips for your Coverly eSIM. Most travelers find their answer in under a minute.
      </p>

      <div class="search-glow" style="max-width: 720px;">
        <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" focusable="false" viewBox="0 0 24 24" class="search-icon" aria-hidden="true" style="position: absolute; left: 22px; top: 50%; transform: translateY(-50%); z-index: 2;">
          <circle cx="11" cy="11" r="7" fill="none" stroke="currentColor" stroke-width="1.6"/>
          <path stroke="currentColor" stroke-linecap="round" stroke-width="1.6" d="m20 20-3.5-3.5"/>
        </svg>
        {{search submit=false instant=settings.instant_search class='searchbar'}}
      </div>

      <div class="quick-chips">
        <span style="font-size: .84rem; opacity: .85; margin-right: 4px;">Popular:</span>
        <a href="#" class="chip">Install on iPhone</a>
        <a href="#" class="chip">No data after install</a>
        <a href="#" class="chip">Check coverage</a>
        <a href="#" class="chip">Request a refund</a>
        <a href="#" class="chip">Multi-country trip</a>
      </div>

      <div class="hero-stats">
        <div class="hero-stat">
          <div class="num">190+</div>
          <div class="lbl">countries covered</div>
        </div>
        <div class="hero-stat">
          <div class="num">~38s</div>
          <div class="lbl">avg. install time</div>
        </div>
        <div class="hero-stat">
          <div class="num">4.8/5</div>
          <div class="lbl">from 218k travelers</div>
        </div>
      </div>
    </div>
  </section>

  <div class="wrap">
    <div class="status-banner">
      <div class="dot-wrap"><span class="status-dot"></span></div>
      <div>
        <h4>All Coverly systems operational</h4>
        <p>Activations, top-ups, and our partner networks are all running normally.</p>
      </div>
      <a href="#" class="linkish">View status page
        <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" focusable="false" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
          <path d="M5 12h14"/><path d="m13 6 6 6-6 6"/>
        </svg>
      </a>
    </div>
  </div>

  <section class="block">
    <div class="wrap">
      <div class="section-head">
        <div>
          <div class="eyebrow">Browse by topic</div>
          <h2>Find your answer faster.</h2>
        </div>
      </div>
      <div class="cat-grid">
        {{#each categories}}
          <a href="{{url}}" class="cat-card">
            <div class="icon">
              {{#is id "getting-started"}}
                <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><circle cx="12" cy="12" r="9"/><path d="m16 8-3 6-6 3 3-6 6-3z"/></svg>
              {{else}}{{#is id "installing"}}
                <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><path d="M12 4v12"/><path d="m7 11 5 5 5-5"/><path d="M5 20h14"/></svg>
              {{else}}{{#is id "plans-coverage"}}
                <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><circle cx="12" cy="12" r="9"/><path d="M3 12h18"/><path d="M12 3a13 13 0 0 1 0 18"/><path d="M12 3a13 13 0 0 0 0 18"/></svg>
              {{else}}{{#is id "account-billing"}}
                <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><path d="M3 7a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2v10a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2z"/><path d="M16 12h3"/></svg>
              {{else}}{{#is id "troubleshooting"}}
                <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><path d="M14.7 6.3a4 4 0 0 0 5 5L19 12l-7 7a3 3 0 1 1-4-4l7-7z"/></svg>
              {{else}}{{#is id "travel-tips"}}
                <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><rect x="3" y="7" width="18" height="13" rx="2"/><path d="M8 7V5a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"/><path d="M12 11v5"/></svg>
              {{else}}
                <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><path d="M14 3H6a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V9z"/><path d="M14 3v6h6"/></svg>
              {{/is}}{{/is}}{{/is}}{{/is}}{{/is}}{{/is}}
            </div>
            <h3>{{name}}</h3>
            <p>{{excerpt description}}</p>
            <div class="meta">
              <span>{{sections.length}} sections</span>
              <span class="arrow">→</span>
            </div>
          </a>
        {{/each}}
      </div>
    </div>
  </section>

  {{#if promoted_articles}}
    <section class="block-tight">
      <div class="wrap">
        <div class="section-head">
          <div>
            <div class="eyebrow">Trending this week</div>
            <h2>What other travelers are reading.</h2>
          </div>
        </div>
        <div class="trending-grid">
          {{#each promoted_articles}}
            <a href="{{url}}" class="trending-item">
              <div class="item-body">
                <h4>{{title}}</h4>
                <div class="meta">Promoted article</div>
              </div>
            </a>
          {{/each}}
        </div>
      </div>
    </section>
  {{/if}}

  <section class="block-tight">
    <div class="wrap">
      <div class="cta-block">
        <div>
          <div class="eyebrow">Still stuck?</div>
          <h2 class="display">Our humans answer in under an hour.</h2>
          <p>
            Real travelers staffing the inbox 24/7 from Lisbon, Bangkok, and Mexico City. They've installed thousands of eSIMs themselves.
          </p>
          {{#link 'new_request' class='submit-btn'}}
            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
              <path d="m22 2-11 11"/><path d="m22 2-7 20-4-9-9-4 20-7z"/>
            </svg>
            <span>Contact support</span>
          {{/link}}
        </div>
        <div class="cta-cards">
          <div class="contact-card">
            <h4>
              <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
                <path d="M21 12a8 8 0 0 1-12 7l-5 1 1-5a8 8 0 1 1 16-3z"/>
              </svg>
              Live chat
            </h4>
            <p>Tap the bubble bottom-right. Median wait: 2 min.</p>
          </div>
          <div class="contact-card">
            <h4>
              <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
                <rect x="3" y="5" width="18" height="14" rx="2"/><path d="m3 7 9 7 9-7"/>
              </svg>
              Email
            </h4>
            <p>help@coverly.com — first reply within 1 hour.</p>
          </div>
        </div>
      </div>
    </div>
  </section>
</main>
```

Curlybars notes:
- `{{#is a b}}…{{else}}…{{/is}}` is the Curlybars equality conditional (Zendesk's docs call it the `is` helper). Nested `{{#is}}…{{else}}{{#is}}…{{/is}}{{/is}}` forms a chain; close all of them at the end.
- `{{excerpt description}}` trims `description` to a short preview.
- `{{#link 'new_request' class='submit-btn'}}` is a block helper that wraps its inner content in an anchor pointing at the new-request page.

- [ ] **Step 3: Verify build succeeds**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 4: Commit**

```bash
git add templates/home_page.hbs
git commit -m "feat(home): rewrite home page to Coverly markup with category icons"
```

---

## Phase 4: Category & Section pages

After Phase 4, browsing into any category and any section renders the Coverly editorial layout.

### Task 14: Rewrite `_category.scss`

Covers the category-page chrome (`.page-header`, `.section-block`) and the shared article row markup used by both category and section pages.

**Files:**
- Modify: `styles/_category.scss`

- [ ] **Step 1: Reference**

Read lines 537–595 + 461–495 of `coverly-support-center/project/styles.css` (page header / section block / article row / tag).

- [ ] **Step 2: Replace file contents**

Replace the entire contents of `styles/_category.scss` with:

```scss
.page-header {
  padding: 24px 0 48px;

  .eyebrow {
    font-size: 0.76rem;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: var(--ink-mute);
    margin-bottom: 8px;
  }

  h1 {
    font-family: var(--font-display);
    font-weight: var(--display-weight);
    letter-spacing: var(--display-tracking);
    font-size: clamp(2.2rem, 4.5vw, 3.6rem);
    margin: 8px 0 12px;
    line-height: 1.05;
    max-width: 22ch;
    color: var(--ink);
  }

  .lead,
  .page-header-description {
    font-size: 1.12rem;
    color: var(--ink-soft);
    margin: 0;
    max-width: 60ch;
  }

  .stats {
    display: flex;
    gap: 28px;
    margin-top: 22px;
    font-size: 0.88rem;
    color: var(--ink-mute);
    flex-wrap: wrap;

    strong {
      color: var(--ink);
      font-weight: 600;
    }
  }
}

.section-block,
.section-tree .section {
  background: var(--bg-elev);
  border: 1px solid var(--line);
  border-radius: 18px;
  padding: 28px 32px;
  margin-bottom: 18px;

  h3,
  .section-tree-title {
    font-family: var(--font-display);
    font-weight: var(--display-weight);
    letter-spacing: var(--display-tracking);
    font-size: 1.5rem;
    margin: 0 0 4px;
    color: var(--ink);

    a { color: inherit; }
  }

  .sec-meta {
    font-size: 0.85rem;
    color: var(--ink-mute);
    margin-bottom: 14px;
  }
}

/***** Article list + article row (shared by category, section, search) *****/
.article-list {
  display: flex;
  flex-direction: column;
  list-style: none;
  padding: 0;
  margin: 0;
}

.article-list-item {
  // Zendesk's default article markup emits <li> with an <a class="article-list-link">.
  // Restyle to look like an .article-row.
  display: grid;
  grid-template-columns: 1fr auto;
  gap: 22px;
  align-items: center;
  padding: 18px 0;
  border-bottom: 1px solid var(--line);
  transition: padding .15s;

  &:hover {
    padding-left: 8px;

    .article-list-link { color: var(--primary); }
  }

  .article-list-link {
    font-size: 1.05rem;
    font-weight: 500;
    color: var(--ink);
    transition: color .15s;
  }

  .icon-star,
  .icon-lock {
    color: var(--accent);
  }
}

.article-row {
  display: grid;
  grid-template-columns: 1fr auto;
  gap: 22px;
  align-items: center;
  padding: 18px 0;
  border-bottom: 1px solid var(--line);
  cursor: pointer;
  transition: padding .15s;
  color: inherit;

  &:hover {
    padding-left: 8px;
    text-decoration: none;

    .article-title { color: var(--primary); }
  }

  .article-title {
    font-size: 1.05rem;
    font-weight: 500;
    color: var(--ink);
    transition: color .15s;
  }

  .article-sub {
    font-size: 0.84rem;
    color: var(--ink-mute);
    margin-top: 3px;
    display: inline-flex;
    align-items: center;
    gap: 6px;
  }

  .badges {
    display: flex;
    gap: 6px;
    flex-wrap: wrap;
    max-width: 320px;
    justify-content: flex-end;
  }
}

.see-all-articles {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  margin-top: 14px;
  color: var(--primary);
  font-weight: 600;
  font-size: 0.92rem;

  &:hover { gap: 10px; }
}

/***** Sub-nav (search + breadcrumbs row above category/section pages) *****/
.sub-nav {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 16px 0;
  flex-wrap: wrap;

  .search-container {
    margin-left: auto;
    flex: 0 1 320px;
  }
}
```

- [ ] **Step 3: Verify build succeeds**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 4: Commit**

```bash
git add styles/_category.scss
git commit -m "feat(styles): port Coverly category page + shared article-row"
```

---

### Task 15: Rewrite `templates/category_page.hbs`

**Files:**
- Modify: `templates/category_page.hbs`

- [ ] **Step 1: Reference**

Read `coverly-support-center/project/pages-main.jsx` lines 205–247 (the `<CategoryPage>` component) and `components.jsx` lines 165–178 (the `<ArticleRow>` component).

- [ ] **Step 2: Replace file contents**

Replace the entire contents of `templates/category_page.hbs` with:

```hbs
<div class="container">
  <div class="wrap">
    <div class="sub-nav">
      {{breadcrumbs}}
      <div class="search-container">
        {{search scoped=settings.scoped_kb_search submit=false}}
      </div>
    </div>

    <header class="page-header">
      <div class="eyebrow mono">Category</div>
      <h1>{{category.name}}</h1>
      {{#if category.description}}
        <p class="lead page-header-description">{{category.description}}</p>
      {{/if}}
      <div class="stats">
        <span><strong>{{sections.length}}</strong> sections</span>
        <span>Updated this week</span>
      </div>
    </header>

    <div id="main-content" class="section-tree">
      {{#each sections}}
        <section class="section section-block">
          <h2 class="section-tree-title">
            <a href="{{url}}">{{name}}</a>
          </h2>

          {{#if articles}}
            <ul class="article-list">
              {{#each articles}}
                <li class="article-list-item {{#if promoted}}article-promoted{{/if}}">
                  {{#if promoted}}
                    <svg xmlns="http://www.w3.org/2000/svg" width="12" height="12" focusable="false" viewBox="0 0 12 12" class="icon-star" title="{{t 'promoted'}}" aria-hidden="true">
                      <path fill="currentColor" d="M2.88 11.73c-.19 0-.39-.06-.55-.18a.938.938 0 01-.37-1.01l.8-3L.35 5.57a.938.938 0 01-.3-1.03c.12-.37.45-.63.85-.65L4 3.73 5.12.83c.14-.37.49-.61.88-.61s.74.24.88.6L8 3.73l3.11.17a.946.946 0 01.55 1.68L9.24 7.53l.8 3a.95.95 0 01-1.43 1.04L6 9.88l-2.61 1.69c-.16.1-.34.16-.51.16z"/>
                    </svg>
                  {{/if}}
                  <a href="{{url}}" class="article-list-link">{{title}}</a>
                  {{#if internal}}
                    <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" focusable="false" viewBox="0 0 16 16" class="icon-lock" title="{{t 'internal'}}" aria-hidden="true">
                      <rect width="12" height="9" x="2" y="7" fill="currentColor" rx="1" ry="1"/>
                      <path fill="none" stroke="currentColor" d="M4.5 7.5V4a3.5 3.5 0 017 0v3.5"/>
                    </svg>
                  {{/if}}
                </li>
              {{/each}}
            </ul>

            {{#if more_articles}}
              <a href="{{url}}" class="see-all-articles">
                {{t 'show_all_articles' count=article_count}}
              </a>
            {{/if}}
          {{/if}}
        </section>
      {{/each}}
    </div>
  </div>
</div>
```

Note: keeping `<li>` markup with Zendesk's default `.article-list-item` class so existing translation strings like `show_all_articles` resolve, and so the meta line (which we drop) doesn't need a per-section data fabrication. The new `_category.scss` styles `.article-list-item` to look like a Coverly row.

- [ ] **Step 3: Verify build succeeds**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 4: Commit**

```bash
git add templates/category_page.hbs
git commit -m "feat(category): rewrite category page to Coverly markup"
```

---

### Task 16: Rewrite `_section.scss`

Most styling reuses `_category.scss`; this partial only adds the bits that are section-page-specific.

**Files:**
- Modify: `styles/_section.scss`

- [ ] **Step 1: Replace file contents**

Replace the entire contents of `styles/_section.scss` with:

```scss
// Section page reuses the .page-header, .section-block, .article-list,
// and .article-list-item rules from _category.scss. This partial adds
// the small bits unique to the section page.

.section-list {
  list-style: none;
  padding: 0;
  margin: 0 0 28px;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 12px;
}

.section-list-item a {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  padding: 16px 18px;
  background: var(--bg-elev);
  border: 1px solid var(--line);
  border-radius: 12px;
  color: var(--ink);
  font-weight: 500;
  transition: border-color .15s, transform .15s;

  &:hover {
    border-color: var(--line-strong);
    transform: translateY(-1px);
    text-decoration: none;
  }

  svg { color: var(--ink-mute); }
}

.section-subscribe {
  margin-top: 18px;
}
```

- [ ] **Step 2: Verify build succeeds**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 3: Commit**

```bash
git add styles/_section.scss
git commit -m "feat(styles): port Coverly section page (nested sections list)"
```

---

### Task 17: Rewrite `templates/section_page.hbs`

**Files:**
- Modify: `templates/section_page.hbs`

- [ ] **Step 1: Reference**

Read `coverly-support-center/project/pages-main.jsx` lines 249–281 (the `<SectionPage>` component).

- [ ] **Step 2: Replace file contents**

Replace the entire contents of `templates/section_page.hbs` with:

```hbs
<div class="container">
  <div class="wrap">
    <div class="sub-nav">
      {{breadcrumbs}}
      <div class="search-container">
        {{search scoped=settings.scoped_kb_search submit=false}}
      </div>
    </div>

    <section id="main-content">
      <header class="page-header">
        <div class="eyebrow mono">Section</div>
        <h1>{{section.name}}</h1>
        {{#if section.description}}
          <p class="lead page-header-description">{{section.description}}</p>
        {{/if}}
        <div class="stats">
          <span><strong>{{section.articles.length}}</strong> articles</span>
          <span>In <strong>{{section.category.name}}</strong></span>
        </div>
        {{#if settings.show_follow_section}}
          <div class="section-subscribe">{{subscribe}}</div>
        {{/if}}
      </header>

      {{#if section.sections}}
        <ul class="section-list">
          {{#each section.sections}}
            <li class="section-list-item">
              <a href="{{url}}">
                <span>{{name}}</span>
                <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" focusable="false" viewBox="0 0 16 16" aria-hidden="true">
                  <path fill="none" stroke="currentColor" stroke-linecap="round" stroke-width="2" d="M5 14.5l6.1-6.1c.2-.2.2-.5 0-.7L5 1.5"/>
                </svg>
              </a>
            </li>
          {{/each}}
        </ul>
      {{/if}}

      {{pagination "section.sections"}}

      {{#if section.articles}}
        <div class="section-block">
          <ul class="article-list">
            {{#each section.articles}}
              <li class="article-list-item {{#if promoted}}article-promoted{{/if}}">
                {{#if promoted}}
                  <svg xmlns="http://www.w3.org/2000/svg" width="12" height="12" focusable="false" viewBox="0 0 12 12" class="icon-star" title="{{t 'promoted'}}" aria-hidden="true">
                    <path fill="currentColor" d="M2.88 11.73c-.19 0-.39-.06-.55-.18a.938.938 0 01-.37-1.01l.8-3L.35 5.57a.938.938 0 01-.3-1.03c.12-.37.45-.63.85-.65L4 3.73 5.12.83c.14-.37.49-.61.88-.61s.74.24.88.6L8 3.73l3.11.17a.946.946 0 01.55 1.68L9.24 7.53l.8 3a.95.95 0 01-1.43 1.04L6 9.88l-2.61 1.69c-.16.1-.34.16-.51.16z"/>
                  </svg>
                {{/if}}
                <a href="{{url}}" class="article-list-link">{{title}}</a>
                {{#if internal}}
                  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" focusable="false" viewBox="0 0 16 16" class="icon-lock" title="{{t 'internal'}}" aria-hidden="true">
                    <rect width="12" height="9" x="2" y="7" fill="currentColor" rx="1" ry="1"/>
                    <path fill="none" stroke="currentColor" d="M4.5 7.5V4a3.5 3.5 0 017 0v3.5"/>
                  </svg>
                {{/if}}
              </li>
            {{/each}}
          </ul>
        </div>
      {{/if}}

      {{pagination "section.articles"}}
    </section>
  </div>
</div>
```

- [ ] **Step 3: Verify build succeeds**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 4: Commit**

```bash
git add templates/section_page.hbs
git commit -m "feat(section): rewrite section page to Coverly markup"
```

---

## Phase 5: Article page

### Task 18: Rewrite `_article.scss`

Covers article-page chrome (`.article-page`, `.article-header`, `.article-meta`), the editorial body styling for WYSIWYG output, opt-in `.callout` and `.step-card` classes, the helpful widget, related grid, and the article sidebar (gated by setting at the template level).

**Files:**
- Modify: `styles/_article.scss`

- [ ] **Step 1: Reference**

Read lines 575–727 of `coverly-support-center/project/styles.css` (covers `.article-page`, `.article-header`, `.article-meta`, `.article-body`, `.callout`, `.step-card`, `.helpful`, `.related-grid`, `.related-card`, `.toc`).

- [ ] **Step 2: Replace file contents**

Replace the entire contents of `styles/_article.scss` with:

```scss
/***** Article page layout *****/
.article-page,
.article-container {
  padding: 8px 0 64px;
  display: grid;
  grid-template-columns: 1fr;
  gap: 56px;

  &.with-sidebar {
    grid-template-columns: 240px 1fr;

    @media (max-width: 900px) {
      grid-template-columns: 1fr;
    }
  }
}

.article-header {
  margin-bottom: 36px;

  .eyebrow {
    font-size: 0.76rem;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: var(--ink-mute);
  }

  h1.article-title,
  h1.display {
    font-family: var(--font-display);
    font-weight: var(--display-weight);
    letter-spacing: var(--display-tracking);
    line-height: 1.08;
    font-size: clamp(2rem, 4vw, 3rem);
    margin: 10px 0 18px;
    max-width: 24ch;
    color: var(--ink);
  }
}

.article-meta,
.article-author {
  display: flex;
  gap: 24px;
  flex-wrap: wrap;
  align-items: center;
  padding: 16px 0;
  border-top: 1px solid var(--line);
  border-bottom: 1px solid var(--line);
  font-size: 0.86rem;
  color: var(--ink-mute);

  .meta-item {
    display: inline-flex;
    align-items: center;
    gap: 8px;
  }

  .author {
    display: flex;
    align-items: center;
    gap: 10px;
    color: var(--ink-soft);
  }

  .avatar {
    width: 32px;
    height: 32px;
    border-radius: 999px;
    background: var(--bg-tint);
    color: var(--ink-soft);
    display: grid;
    place-items: center;
    font-weight: 600;
    font-size: 0.8rem;
    overflow: hidden;

    img { width: 100%; height: 100%; object-fit: cover; }
  }

  .meta-group {
    display: inline-flex;
    align-items: center;
    gap: 6px;
  }
}

/***** Sidebar (articles in section) — gated by settings.show_articles_in_section *****/
.article-sidebar {
  .collapsible-sidebar {
    background: var(--bg-elev);
    border: 1px solid var(--line);
    border-radius: 14px;
    padding: 18px 18px;
    position: sticky;
    top: 100px;
  }

  .collapsible-sidebar-title,
  .sidenav-title {
    font-size: 0.74rem;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: var(--ink-mute);
    display: block;
    margin-bottom: 14px;
  }

  ul {
    list-style: none;
    padding: 0;
    margin: 0;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  a.sidenav-item {
    color: var(--ink-soft);
    font-size: 0.92rem;
    transition: color .15s;

    &:hover { color: var(--ink); }

    &.current-article {
      color: var(--primary);
      font-weight: 600;
    }
  }

  .article-sidebar-item {
    margin-top: 14px;
    color: var(--primary);
    font-size: 0.88rem;
  }

  .collapsible-sidebar-toggle {
    display: none;

    @media (max-width: 900px) { display: inline-flex; }
  }
}

/***** Article body *****/
.article-body {
  font-size: 1.06rem;
  line-height: 1.7;
  color: var(--ink);

  h2 {
    font-family: var(--font-display);
    font-weight: var(--display-weight);
    letter-spacing: var(--display-tracking);
    font-size: 1.7rem;
    margin: 44px 0 14px;
    color: var(--ink);
  }

  h3 {
    font-family: var(--font-display);
    font-weight: var(--display-weight);
    letter-spacing: var(--display-tracking);
    font-size: 1.25rem;
    margin: 32px 0 10px;
    color: var(--ink);
  }

  h4 {
    font-family: var(--font-display);
    font-weight: var(--display-weight);
    font-size: 1.1rem;
    margin: 28px 0 8px;
  }

  p {
    margin: 0 0 18px;
    color: var(--ink-soft);
  }

  ol, ul {
    color: var(--ink-soft);
    padding-left: 22px;
    margin: 0 0 18px;
    list-style: revert;
  }

  li { margin: 0 0 8px; }

  blockquote {
    border-left: 3px solid var(--accent);
    padding: 8px 16px;
    margin: 18px 0;
    color: var(--ink-soft);
    font-style: italic;
  }

  a {
    color: var(--primary);
    border-bottom: 1px solid color-mix(in oklab, var(--primary) 35%, transparent);

    &:hover { border-bottom-color: var(--primary); }
  }

  code {
    font-family: var(--font-mono);
    background: var(--bg-tint);
    padding: 2px 6px;
    border-radius: 4px;
    font-size: 0.88em;
  }

  pre {
    background: var(--bg-tint);
    border: 1px solid var(--line);
    border-radius: 8px;
    padding: 14px 16px;
    overflow-x: auto;
    font-family: var(--font-mono);
    font-size: 0.88em;
    line-height: 1.55;

    code {
      background: transparent;
      padding: 0;
    }
  }

  img, figure {
    margin: 22px 0;
    border-radius: 10px;
  }

  table {
    width: 100%;
    border-collapse: collapse;
    margin: 22px 0;

    th, td {
      padding: 10px 14px;
      border-bottom: 1px solid var(--line);
      text-align: left;
    }

    th { color: var(--ink); font-weight: 600; }
    td { color: var(--ink-soft); }
  }

  /***** Opt-in classes — authors paste raw HTML with these classes to get styled treatment *****/
  .callout {
    background: var(--bg-tint);
    border-left: 3px solid var(--accent);
    border-radius: 6px;
    padding: 16px 18px;
    margin: 20px 0;
    font-size: 0.96rem;

    strong { color: var(--ink); }
  }

  .step-card {
    background: var(--bg-elev);
    border: 1px solid var(--line);
    border-radius: 12px;
    padding: 18px 20px;
    margin: 14px 0;
    display: flex;
    gap: 18px;

    .step-num {
      font-family: var(--font-display);
      font-size: 1.6rem;
      color: var(--accent);
      width: 40px;
      flex-shrink: 0;
      font-weight: var(--display-weight);
    }

    .step-body {
      color: var(--ink-soft);

      strong {
        color: var(--ink);
        display: block;
        margin-bottom: 4px;
        font-weight: 600;
      }
    }
  }
}

/***** Attachments *****/
.article-attachments {
  background: var(--bg-elev);
  border: 1px solid var(--line);
  border-radius: 12px;
  padding: 18px 20px;
  margin: 24px 0;

  .attachments {
    list-style: none;
    padding: 0;
    margin: 0;
  }

  .attachment-item {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 8px 0;
    border-bottom: 1px solid var(--line);
    color: var(--ink-soft);

    &:last-child { border-bottom: none; }

    a { color: var(--primary); }
    .attachment-icon { color: var(--ink-mute); }
    .meta-data { color: var(--ink-mute); font-size: 0.82rem; }
  }
}

/***** Helpful widget (Zendesk vote helper, restyled) *****/
.article-votes,
.helpful {
  margin: 56px 0 40px;
  padding: 32px;
  background: var(--bg-elev);
  border: 1px solid var(--line);
  border-radius: 18px;
  text-align: center;

  h2.article-votes-question,
  h4 {
    font-family: var(--font-display);
    font-weight: var(--display-weight);
    letter-spacing: var(--display-tracking);
    font-size: 1.5rem;
    margin: 0 0 14px;
    color: var(--ink);
  }

  p.muted {
    margin: 0 0 18px;
    font-size: 0.9rem;
    color: var(--ink-mute);
  }

  .article-votes-controls,
  .helpful-buttons {
    display: flex;
    gap: 12px;
    justify-content: center;
  }

  .article-vote,
  button.button {
    background: var(--bg-elev);
    border: 1px solid var(--line-strong);
    color: var(--ink);
    padding: 12px 28px;
    border-radius: 999px;
    font-weight: 500;
    font-size: 0.95rem;
    transition: all .15s;
    display: inline-flex;
    align-items: center;
    gap: 8px;
    cursor: pointer;

    &:hover { border-color: var(--ink-soft); }

    &.button-primary {
      background: var(--primary);
      color: var(--primary-ink);
      border-color: var(--primary);
    }
  }

  .article-votes-count,
  .helpful-feedback {
    display: block;
    margin-top: 16px;
    font-size: 0.92rem;
    color: var(--ink-soft);
  }
}

/***** Related & recent article grids *****/
.related-grid {
  // Coverly's direct grid (used when we author markup ourselves).
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 14px;
  margin-top: 28px;

  @media (max-width: 700px) { grid-template-columns: 1fr; }
}

.article-relatives {
  // Zendesk's {{recent_articles}} and {{related_articles}} helpers each emit a
  // labelled <ul> here. Stack them vertically and grid the items within each ul.
  margin-top: 28px;
  display: flex;
  flex-direction: column;
  gap: 32px;
}

.recent-article-list,
.related-article-list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 14px;

  @media (max-width: 700px) { grid-template-columns: 1fr; }
}

.related-card,
.recent-article-list-item,
.related-article-list-item {
  background: var(--bg-elev);
  border: 1px solid var(--line);
  border-radius: 14px;
  padding: 18px;
  cursor: pointer;
  transition: border-color .15s, transform .15s;
  color: inherit;

  &:hover {
    border-color: var(--line-strong);
    transform: translateY(-2px);
    text-decoration: none;
  }

  .related-eyebrow {
    font-size: 0.76rem;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: var(--ink-mute);
    margin-bottom: 6px;
  }

  h4 {
    margin: 0;
    font-size: 1rem;
    font-weight: 500;
    color: var(--ink);
  }

  a {
    color: inherit;
    font-size: 1rem;
    font-weight: 500;

    &:hover { text-decoration: none; color: var(--primary); }
  }
}

/***** Share buttons (Zendesk {{share}} helper) *****/
.article-share {
  margin: 28px 0;

  a {
    color: var(--ink-mute);
    transition: color .15s;

    &:hover { color: var(--primary); }
  }
}

/***** Article footer container *****/
.article-footer {
  margin-top: 32px;
  display: flex;
  align-items: center;
  gap: 18px;
  flex-wrap: wrap;
  padding-top: 18px;
  border-top: 1px solid var(--line);
}
```

- [ ] **Step 3: Verify build succeeds**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 4: Commit**

```bash
git add styles/_article.scss
git commit -m "feat(styles): port Coverly article page (body, helpful, related)"
```

---

### Task 19: Rewrite `templates/article_page.hbs`

Removes the comments block entirely, gates the sidebar behind `settings.show_articles_in_section`, drops the "Return to top" link, and uses Coverly's narrow-column layout for the body.

**Files:**
- Modify: `templates/article_page.hbs`

- [ ] **Step 1: Reference**

Read `coverly-support-center/project/pages-main.jsx` lines 364–468 (the `<ArticlePage>` component).

- [ ] **Step 2: Replace file contents**

Replace the entire contents of `templates/article_page.hbs` with:

```hbs
<div class="container">
  <div class="{{#if settings.show_articles_in_section}}wrap{{else}}wrap-narrow{{/if}}">
    <div class="sub-nav">
      {{breadcrumbs}}
      <div class="search-container">
        {{search scoped=settings.scoped_kb_search submit=false}}
      </div>
    </div>

    <div class="article-container {{#if settings.show_articles_in_section}}with-sidebar article-page{{else}}article-page{{/if}}" id="article-container">
      {{#if settings.show_articles_in_section}}
        <aside class="article-sidebar" aria-labelledby="section-articles-title">
          <div class="collapsible-sidebar">
            <button type="button" class="collapsible-sidebar-toggle" aria-labelledby="section-articles-title" aria-expanded="false">
              <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" focusable="false" viewBox="0 0 12 12" aria-hidden="true" class="collapsible-sidebar-toggle-icon chevron-icon">
                <path fill="none" stroke="currentColor" stroke-linecap="round" d="M3 4.5l2.6 2.6c.2.2.5.2.7 0L9 4.5"/>
              </svg>
              <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" focusable="false" viewBox="0 0 12 12" aria-hidden="true" class="collapsible-sidebar-toggle-icon x-icon">
                <path stroke="currentColor" stroke-linecap="round" d="M3 9l6-6m0 6L3 3"/>
              </svg>
            </button>
            <span id="section-articles-title" class="collapsible-sidebar-title sidenav-title">
              {{t 'articles_in_section'}}
            </span>
            <div class="collapsible-sidebar-body">
              <ul>
                {{#each section.articles}}
                  <li>
                    <a href="{{url}}"
                       class="sidenav-item {{#is id ../article.id}}current-article{{/is}}"
                       {{#is id ../article.id}}aria-current="page"{{/is}}>
                       {{title}}
                    </a>
                  </li>
                {{/each}}
              </ul>
              {{#if section.more_articles}}
                <a href="{{section.url}}" class="article-sidebar-item">{{t 'see_more'}}</a>
              {{/if}}
            </div>
          </div>
        </aside>
      {{/if}}

      <article id="main-content" class="article">
        <header class="article-header">
          <div class="eyebrow mono">{{section.category.name}} · {{section.name}}</div>
          <h1 title="{{article.title}}" class="article-title display">
            {{article.title}}
            {{#if article.internal}}
              <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" focusable="false" viewBox="0 0 16 16" class="icon-lock" title="{{t 'internal'}}" aria-hidden="true">
                <rect width="12" height="9" x="2" y="7" fill="currentColor" rx="1" ry="1"/>
                <path fill="none" stroke="currentColor" d="M4.5 7.5V4a3.5 3.5 0 017 0v3.5"/>
              </svg>
            {{/if}}
          </h1>

          <div class="article-meta article-author">
            {{#if settings.show_article_author}}
              <span class="author">
                <span class="avatar article-avatar">
                  <img src="{{article.author.avatar_url}}" alt="" class="user-avatar"/>
                </span>
                {{#link 'user_profile' id=article.author.id}}
                  {{article.author.name}}
                {{/link}}
              </span>
            {{/if}}

            <span class="meta-item meta-group">
              {{#is article.created_at article.edited_at}}
                <span class="meta-data">{{date article.created_at timeago=true}}</span>
              {{else}}
                <span class="meta-data">{{date article.edited_at timeago=true}}</span>
                <span class="meta-data">{{t 'updated'}}</span>
              {{/is}}
            </span>

            {{#if settings.show_follow_article}}
              <span class="article-subscribe" style="margin-left: auto;">{{subscribe}}</span>
            {{/if}}
          </div>
        </header>

        <section class="article-info">
          <div class="article-content">
            <div class="article-body">{{article.body}}</div>

            {{#if (compare article.content_tags.length ">" 0)}}
              <section class="content-tags">
                <p>{{t 'content_tags_label'}}</p>
                <ul class="content-tag-list">
                  {{#each article.content_tags}}
                    <li class="content-tag-item" data-content-tag-id="{{id}}">
                      {{#link "search_result" content_tag_id=id}}
                        {{name}}
                      {{/link}}
                    </li>
                  {{/each}}
                </ul>
              </section>
            {{/if}}

            {{#if attachments}}
              <div class="article-attachments">
                <ul class="attachments">
                  {{#each attachments}}
                    <li class="attachment-item">
                      <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" focusable="false" viewBox="0 0 16 16" class="attachment-icon" aria-hidden="true">
                        <path fill="none" stroke="currentColor" stroke-linecap="round" d="M9.5 4v7.7c0 .8-.7 1.5-1.5 1.5s-1.5-.7-1.5-1.5V3C6.5 1.6 7.6.5 9 .5s2.5 1.1 2.5 2.5v9c0 1.9-1.6 3.5-3.5 3.5S4.5 13.9 4.5 12V4"/>
                      </svg>
                      <a href="{{url}}" target="_blank">{{name}}</a>
                      <div class="attachment-meta meta-group">
                        <span class="attachment-meta-item meta-data">{{size}}</span>
                        <a href="{{url}}" target="_blank" class="attachment-meta-item meta-data">{{t 'download'}}</a>
                      </div>
                    </li>
                  {{/each}}
                </ul>
              </div>
            {{/if}}
          </div>
        </section>

        <footer>
          {{#if settings.show_article_sharing}}
            <div class="article-footer">
              <div class="article-share">{{share}}</div>
            </div>
          {{/if}}

          {{#with article}}
            <div class="article-votes helpful">
              <h2 class="article-votes-question" id="article-votes-label">{{t 'was_this_article_helpful'}}</h2>
              <div class="article-votes-controls helpful-buttons" role="group" aria-labelledby="article-votes-label">
                {{vote 'up'   class='button article-vote article-vote-up'   selected_class="button-primary"}}
                {{vote 'down' class='button article-vote article-vote-down' selected_class="button-primary"}}
              </div>
              <small class="article-votes-count helpful-feedback">
                {{vote 'label' class='article-vote-label'}}
              </small>
            </div>
          {{/with}}

          <div class="article-more-questions">
            {{request_callout}}
          </div>
        </footer>

        <div class="article-relatives">
          {{#if settings.show_recently_viewed_articles}}
            {{recent_articles}}
          {{/if}}
          {{#if settings.show_related_articles}}
            {{related_articles}}
          {{/if}}
        </div>
      </article>
    </div>
  </div>
</div>
```

Key removals from the original Copenhagen template:
- Entire `{{#if settings.show_article_comments}}…{{/if}}` block — comments dropped per spec
- `.article-return-to-top` block — not in Coverly design
- "Submit a request" callout link in the article footer is preserved via `{{request_callout}}`

- [ ] **Step 3: Verify build succeeds**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 4: Commit**

```bash
git add templates/article_page.hbs
git commit -m "feat(article): rewrite article page, drop comments, gate sidebar"
```

---

## Phase 6: Cleanup, manifest, docs, verification

### Task 20: Trim `manifest.json`

Remove color, font, and now-irrelevant settings.

**Files:**
- Modify: `manifest.json`

- [ ] **Step 1: Replace file contents**

Replace the entire contents of `manifest.json` with:

```json
{
  "name": "Coverly",
  "author": "Nathan Catania",
  "version": "4.41.0",
  "api_version": 4,
  "default_locale": "en-us",
  "settings": [
    {
      "label": "brand_group_label",
      "variables": [
        {
          "identifier": "logo",
          "type": "file",
          "description": "logo_description",
          "label": "logo_label"
        },
        {
          "identifier": "show_brand_name",
          "type": "checkbox",
          "description": "show_brand_name_description",
          "label": "show_brand_name_label",
          "value": true
        },
        {
          "identifier": "favicon",
          "type": "file",
          "description": "favicon_description",
          "label": "favicon_label"
        }
      ]
    },
    {
      "label": "images_group_label",
      "variables": [
        {
          "identifier": "homepage_background_image",
          "type": "file",
          "description": "homepage_background_image_description",
          "label": "homepage_background_image_label"
        }
      ]
    },
    {
      "label": "search_group_label",
      "variables": [
        {
          "identifier": "instant_search",
          "type": "checkbox",
          "description": "instant_search_description",
          "label": "instant_search_label",
          "value": true
        },
        {
          "identifier": "scoped_kb_search",
          "type": "checkbox",
          "description": "scoped_knowledge_base_search_description",
          "label": "scoped_knowledge_base_search_label_v2",
          "value": true
        }
      ]
    },
    {
      "label": "article_page_group_label",
      "variables": [
        {
          "identifier": "show_articles_in_section",
          "type": "checkbox",
          "description": "articles_in_section_description",
          "label": "articles_in_section_label",
          "value": true
        },
        {
          "identifier": "show_article_author",
          "type": "checkbox",
          "description": "article_author_description",
          "label": "article_author_label",
          "value": true
        },
        {
          "identifier": "show_follow_article",
          "type": "checkbox",
          "description": "follow_article_description",
          "label": "follow_article_label",
          "value": true
        },
        {
          "identifier": "show_recently_viewed_articles",
          "type": "checkbox",
          "description": "recently_viewed_articles_description",
          "label": "recently_viewed_articles_label",
          "value": true
        },
        {
          "identifier": "show_related_articles",
          "type": "checkbox",
          "description": "related_articles_description",
          "label": "related_articles_label",
          "value": true
        },
        {
          "identifier": "show_article_sharing",
          "type": "checkbox",
          "description": "article_sharing_description",
          "label": "article_sharing_label",
          "value": true
        }
      ]
    },
    {
      "label": "section_page_group_label",
      "variables": [
        {
          "identifier": "show_follow_section",
          "type": "checkbox",
          "description": "follow_section_description",
          "label": "follow_section_label",
          "value": true
        }
      ]
    },
    {
      "label": "show_suggested_articles_label",
      "variables": [
        {
          "identifier": "show_suggested_articles",
          "type": "checkbox",
          "description": "show_suggested_articles_in_new_request_form_description",
          "label": "show_suggested_articles_label",
          "value": true
        }
      ]
    }
  ]
}
```

- [ ] **Step 2: Verify build succeeds**

Run: `yarn build`
Expected: clean build. The `$brand_color` and `$text_color` vars are now hardcoded in `_variables.scss` (Task 4), so removing them from `manifest.json` does not break SCSS compilation.

- [ ] **Step 3: Commit**

```bash
git add manifest.json
git commit -m "chore(manifest): trim to Coverly-relevant settings"
```

---

### Task 21: Trim `translations.yml`

Remove translation entries for the manifest settings deleted in Task 20.

**Files:**
- Modify: `translations.yml`

- [ ] **Step 1: Identify keys to remove**

The deleted manifest settings reference these translation keys. Search for each and remove its entire `- translation:` block:

- `brand_color_description`, `brand_color_label`
- `brand_text_color_description`, `brand_text_color_label`
- `text_color_description`, `text_color_label`
- `link_color_description`, `link_color_label`
- `hover_link_color_description`, `hover_link_color_label`
- `visited_link_color_description`, `visited_link_color_label`
- `background_color_description`, `background_color_label`
- `heading_font_description`, `heading_font_label`
- `text_font_description`, `text_font_label`
- `colors_group_label`
- `fonts_group_label`
- `home_page_group_label`
- `recent_activity_description`, `recent_activity_label`
- `article_comments_description`, `article_comments_label`
- `community_background_image_description`, `community_background_image_label`
- `community_image_description`, `community_image_label`
- `service_catalog_hero_image_description`, `service_catalog_hero_image_label`
- `scoped_community_search_description`, `scoped_community_search_label`
- `community_post_group_label`
- `follow_post_description`, `follow_post_label`
- `post_sharing_description`, `post_sharing_label`
- `community_topic_group_label`
- `follow_topic_description`, `follow_topic_label`

- [ ] **Step 2: Edit the file**

Open `translations.yml` and remove each block matching `key: "txt.help_center_copenhagen_theme.<name>"` for the names above. Each block is a `- translation:` entry spanning 4–6 lines. Preserve YAML structure (no trailing commas, consistent indentation).

- [ ] **Step 3: Verify YAML is still valid**

Run:
```bash
python3 -c "import yaml; yaml.safe_load(open('translations.yml'))" && echo "YAML OK"
```
Expected: `YAML OK`. If you get a parse error, find the broken indentation and fix.

- [ ] **Step 4: Verify build succeeds**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 5: Commit**

```bash
git add translations.yml
git commit -m "chore(translations): drop entries for removed manifest settings"
```

---

### Task 22: Trim `styles/index.scss`

Drop the imports that are no longer used (no template references them). Keep partials needed by untouched templates.

**Files:**
- Modify: `styles/index.scss`

- [ ] **Step 1: Audit which partials are still referenced**

Run:
```bash
grep -rEl --include='*.hbs' 'class="[^"]*(community|community_badge|community-badge|post-|striped-list|recent-activity|notifications|user-profile|sub-nav|content-tag|wysiwyg-|upload-dropzone|metadata-|summary-|service-catalog|approval-)' templates/
```
This identifies the classes referenced by untouched templates. Keep partials matching those classes.

- [ ] **Step 2: Replace file contents**

Replace `styles/index.scss` with the trimmed set. Keep the imports below; remove any not listed:

```scss
@import "normalize";
@import "variables";
@import "mixins";
@import "base";

// Shell — used on every page
@import "header";
@import "footer";

// Focus templates
@import "hero";
@import "home-page";
@import "breadcrumbs";
@import "page_header";
@import "sub-nav";
@import "category";
@import "section";
@import "article";

// Untouched templates rely on these — keep for compatibility
@import "buttons";
@import "split_button";
@import "tables";
@import "forms";
@import "user";
@import "search";
@import "blocks";
@import "recent-activity";
@import "attachments";
@import "share";
@import "vote";
@import "actions";
@import "community";
@import "striped_list";
@import "status-label";
@import "post";
@import "community_badge";
@import "collapsible-nav";
@import "collapsible-sidebar";
@import "my-activities";
@import "request";
@import "pagination";
@import "metadata";
@import "user-profiles";
@import "search_results";
@import "notifications";
@import "dropdowns";
@import "content-tags";
@import "wysiwyg";
@import "upload_dropzone";
@import "summary";
@import "service_catalog";
@import "comments";
```

Note: `comments` is kept even though the template removed the comments block — the partial may be referenced by community-post pages.

- [ ] **Step 3: Verify build succeeds**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 4: Commit**

```bash
git add styles/index.scss
git commit -m "chore(styles): regroup index.scss imports by concern"
```

---

### Task 23: Document conventions in `AGENTS.md`

Note the category-icon convention and the `.callout`/`.step-card` opt-in classes.

**Files:**
- Modify: `AGENTS.md`

- [ ] **Step 1: Append a new section**

Add this section to the end of `AGENTS.md`:

```markdown
## Coverly theme conventions

This is a Coverly-branded Help Center theme. A few conventions are worth knowing when authoring or extending:

### Category icons

The home page renders a category icon next to each `.cat-card`. Icons are inline SVG, keyed off the category slug, in `templates/home_page.hbs`. The current mapping covers:

- `getting-started` → compass
- `installing` → download
- `plans-coverage` → globe
- `account-billing` → wallet
- `troubleshooting` → wrench
- `travel-tips` → suitcase
- *any other slug* → generic document icon (fallback)

To add a new icon, edit the `{{#is id "…"}}` chain in `home_page.hbs` and add a new `{{else}}{{#is id "your-slug"}}…<svg>…{{/is}}` branch before the fallback.

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
- `assets/umbrellas.jpg` — default hero background, referenced from `styles/_hero.scss`

### Dark theme is hardcoded

`manifest.json` no longer exposes color or font settings. The palette and typography are baked into `styles/_variables.scss` (SCSS) and `styles/_base.scss` (CSS custom properties on `:root`). Adjust both files together when changing tokens.
```

- [ ] **Step 2: Verify**

Read the file to confirm the section was appended cleanly.

- [ ] **Step 3: Commit**

```bash
git add AGENTS.md
git commit -m "docs: document Coverly theme conventions (icons, callouts, tokens)"
```

---

### Task 24: Final verification

End-to-end check that everything builds and renders before declaring complete.

- [ ] **Step 1: Clean build**

Run:
```bash
yarn build
```
Expected: completes without errors. Both `script.js` and `style.css` are regenerated.

- [ ] **Step 2: Check the produced CSS**

Run:
```bash
grep -c "var(--primary)" style.css
grep -c "var(--bg)" style.css
grep -c "umbrellas.jpg" style.css
```
Expected: positive integers for all three (tokens compiled in, hero image referenced).

- [ ] **Step 3: Lint source**

Run:
```bash
yarn eslint src
```
Expected: passes (we didn't change `src/`, so this should be a no-op).

- [ ] **Step 4: Run existing tests**

Run:
```bash
yarn test
```
Expected: passes (we didn't change `src/`, so React-module tests are unaffected).

- [ ] **Step 5: Launch local preview**

Run:
```bash
yarn zcli themes:preview
```

The CLI will prompt for Zendesk authentication. **STOP HERE and ask the user to log in interactively** (per `AGENTS.md`):

> "The Zendesk preview needs you to authenticate. Run `! zcli login -i` in this conversation if you haven't yet, then re-run preview."

Once preview is up, the CLI prints a URL like `https://glean-73565.zendesk.com/hc/admin/local_preview/start`.

- [ ] **Step 6: Visual verification — Firefox only (per AGENTS.md constraint)**

Open Firefox and visit the preview URL. Verify each:

| Page | Check |
|---|---|
| Home | Dark hero with umbrella photo, eyebrow chip with pulsing dot, display headline with italic accent, animated search-glow, popular chips, 3 hero stats, status banner overlapping seam, category grid (with icons where slugs match), trending grid (if promoted articles), contact CTA |
| One category | `.eyebrow.mono` "Category", display heading, lead, section blocks with article rows |
| One section | Same chrome as category, single section's articles |
| One article (with `show_articles_in_section`=true) | Sidebar visible on left, narrow body, display heading, eyebrow with category·section, author/meta row, body content styled, helpful widget at bottom |
| One article (with `show_articles_in_section`=false) | No sidebar, narrow-column layout |
| One out-of-scope page (e.g., a request form via `/hc/requests/new`) | Header/footer match new design, form is still functional, no contrast disasters |

- [ ] **Step 7: Report results**

If any visual issue requires further work, capture the specific class and partial, then create follow-up issues. Do not patch ad-hoc — escalate to the user for a triage decision.

- [ ] **Step 8: Final commit (only if there are uncommitted changes)**

Most likely there are none at this point — each task already committed. If there's a small tweak from Step 6 verification (e.g., a contrast fix), commit it separately:

```bash
git add <file>
git commit -m "fix(<scope>): <one-line description>"
```

---

## Done

All 24 tasks complete means:
- `coverly-support-center/` is gitignored (Task 1)
- Logo + hero photo packaged as assets (Task 2)
- Google Fonts loaded (Task 3)
- SCSS variables + base tokens defined (Tasks 4–5)
- Header + footer rewritten (Tasks 6–9)
- Hero, home-page partials, breadcrumbs, home template done (Tasks 10–13)
- Category + section partials and templates done (Tasks 14–17)
- Article partial and template done (Tasks 18–19)
- Manifest + translations trimmed (Tasks 20–21)
- styles/index.scss reviewed (Task 22)
- AGENTS.md documents conventions (Task 23)
- Build + preview verified (Task 24)

Hand back to the user with: "Implementation complete. Visual verification passed for home / category / section / article. Out-of-scope pages render with new chrome and inherited palette. See commit history for the full step-by-step trail."
