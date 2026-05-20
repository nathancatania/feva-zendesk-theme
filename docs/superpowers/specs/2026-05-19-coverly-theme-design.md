# Coverly Help Center Theme — Design Spec

**Date**: 2026-05-19
**Repo**: `feva-zendesk-theme` (fork of Zendesk Copenhagen)
**Source design**: `coverly-support-center/` (Claude Design handoff bundle, locked to dark theme + magazine font pair + photo hero)
**Status**: Approved for implementation planning

## Goal

Migrate the Coverly Support Center design (an HTML/CSS/JS prototype produced via Claude Design) into the Copenhagen-based Zendesk Help Center theme in this repo. Replicate the design's visual treatment for the **home, category, section, and article** templates. Header and footer get rewritten to match Coverly's shell. All other pages keep their existing Copenhagen templates but inherit the new color palette and fonts so the theme reads as one product.

## Non-goals

- Restyling request forms, service catalog, approval requests, community, profile, search results, error, or new-community-post pages. These keep Copenhagen templates and React-module styling.
- Restyling the React modules in `src/modules/` (new-request-form, request-list, service-catalog, approval-requests, flash-notifications, ticket-fields). They keep their Garden-themed look.
- Replicating the Coverly bundle's Coverage and Contact pages — eSIM-specific UX that doesn't map to Zendesk Help Center concepts. The home page's "Coverage lookup" widget is dropped.
- Replicating Coverly's custom feedback textarea on the article page (Zendesk's `{{vote}}` helper is what we have).
- Custom-field-driven minutes-to-read, view counts, or "% helpful" percentages on article rows. Those fields don't exist in Zendesk Help Center; substitute with edited-at timeago.
- A full dark-mode audit of every Copenhagen legacy partial. We patch contrast on out-of-scope pages opportunistically, not exhaustively.
- Automated tests for the new templates. The existing Jest suite covers React modules only; Curlybars templates aren't unit-testable in this repo's setup. Verification is manual via local preview.

## Architecture

Approach: **convert Coverly's CSS into focused SCSS partials, rewrite the four focus templates plus header/footer/document_head, leave React modules untouched.** Tokens are exposed as CSS custom properties on `:root`, so untouched partials and React modules inherit the palette automatically.

### File map — changes

| Path | Action |
|---|---|
| `styles/_variables.scss` | Rewrite — Coverly palette as SCSS vars, rebind `$brand_color` / `$text_color` / `$link_color` / `$background_color` / `$button-color` etc. so legacy partials inherit new palette |
| `styles/_base.scss` | Rewrite — `:root` CSS-custom-property block with full Coverly token set; global `body`, helpers (`.display`, `.mono`, `.muted`, `.soft`, `.wrap`, `.wrap-narrow`, `.tag`, `.link-u`, `.status-chip`, `.status-dot`, `@keyframes pulse`) |
| `styles/_header.scss` | Rewrite — `.header`, `.header-inner`, `.brand`, `.nav`, `.pill`, mobile drawer adapted to dark surface |
| `styles/_footer.scss` | Rewrite — 4-column grid (brand+blurb / Help / Coverly / Legal), bottom bar with copyright + status chip + language selector |
| `styles/_hero.scss` | Rewrite — `.hero`, `.hero[data-style="photo"]`, eyebrow chip, display heading, lead, `.search-glow` conic-gradient animation, `.searchbar`, `.quick-chips`, `.hero-stats` |
| `styles/_home-page.scss` | Rewrite — `.status-banner`, `.cat-grid`, `.cat-card`, `.trending-grid`, `.trending-item`, contact CTA block, `.section-head`, `.block`, `.block-tight` |
| `styles/_category.scss` | Rewrite — `.page-header`, `.stats`, `.section-block`, `.article-list`, `.article-row` |
| `styles/_section.scss` | Rewrite — shares partials with category; section-page-specific bits |
| `styles/_article.scss` | Rewrite — `.article-page`, `.article-header`, `.article-meta`, `.article-body` (WYSIWYG output + opt-in `.callout` + `.step-card`), `.helpful` (styled `{{vote}}`), `.related-grid`, `.related-card`, sidebar (`.article-sidebar`, gated by setting) |
| `styles/_breadcrumbs.scss` | Rewrite — `.crumbs` |
| `styles/index.scss` | Trim — keep partials needed by out-of-scope pages (forms, comments, requests, etc.) so they still render; remove obviously-dead partials |
| `templates/document_head.hbs` | Add Google Fonts links (Newsreader, Manrope, JetBrains Mono); keep existing import-map + flash-notifications bootstrap |
| `templates/header.hbs` | Rewrite — Coverly header markup, drop "Ask AI" link, keep skip-navigation, user dropdown, mobile drawer |
| `templates/footer.hbs` | Rewrite — Coverly footer markup, keep language selector |
| `templates/home_page.hbs` | Rewrite — hero / status banner / categories grid / trending grid / contact CTA |
| `templates/category_page.hbs` | Rewrite — page header + section blocks with article rows |
| `templates/section_page.hbs` | Rewrite — page header + article rows |
| `templates/article_page.hbs` | Rewrite — header + body + helpful + related; sidebar gated by `settings.show_articles_in_section`; comments block removed |
| `manifest.json` | Trim — remove color/font settings, remove `show_article_comments`, remove `show_recent_activity`, remove community/service-catalog imagery settings |
| `translations.yml` | Trim — drop labels/descriptions for removed settings |
| `assets/coverly-icon.png` | Add — packaged logo asset |
| `assets/umbrellas.jpg` | Add — packaged hero background |
| `.gitignore` | Add `coverly-support-center/` (reference bundle, not part of shipped theme) |

### File map — untouched

- `src/`, `src/modules/`, `bin/`, `__mocks__/`, `jest.config.mjs`, `jest.setup.js`, `rollup.config.mjs`, `zass.mjs`, `generate-import-map.mjs`, `package.json`, `tsconfig.json`
- `templates/`: `approval_request_*.hbs`, `community_*.hbs`, `contributions_page.hbs`, `error_page.hbs`, `new_community_post_page.hbs`, `new_request_page.hbs`, `request_page.hbs`, `requests_page.hbs`, `search_results.hbs`, `section_page.hbs`'s React-rendered children, `service_*.hbs`, `subscriptions_page.hbs`, `user_profile_page.hbs`
- All untouched legacy SCSS partials (`_buttons`, `_forms`, `_comments`, `_pagination`, `_dropdowns`, `_actions`, `_attachments`, `_share`, `_vote`, `_community`, `_post`, etc.) — they keep compiling and inherit the new palette via the `$brand_color` / `$text_color` rebinding in `_variables.scss`

## Design tokens

A single `:root` block in `styles/_base.scss` defines every Coverly token. No `[data-theme]` or `[data-font-pair]` switching — we ship one theme.

```css
:root {
  /* Brand */
  --brand-navy:      #1F3A6E;
  --brand-cobalt:    #2E66C7;
  --brand-sky:       #6FA3E8;
  --brand-mint:      #5BE0B0;
  --brand-mint-deep: #2BB089;

  /* Surface */
  --bg:           #0C172F;
  --bg-elev:      #142446;
  --bg-tint:      #1B2C53;
  --ink:          #F2F4F9;
  --ink-soft:     #B6C3DD;
  --ink-mute:     #6E7FA4;
  --line:         #25366A;
  --line-strong:  #38498A;

  /* Action */
  --primary:      #5BE0B0;
  --primary-ink:  #0B2A22;
  --accent:       #6FA3E8;
  --accent-ink:   #0C172F;

  /* Hero overlay & shadow */
  --hero-overlay: linear-gradient(180deg, rgba(8, 20, 45, 0.6) 0%, rgba(8, 20, 45, 0.85) 100%);
  --shadow-card:  0 1px 0 rgba(0,0,0,0.4), 0 12px 32px -16px rgba(0,0,0,0.6);

  /* Type */
  --font-display:     "Newsreader", "Georgia", serif;
  --font-body:        "Manrope", system-ui, sans-serif;
  --font-mono:        "JetBrains Mono", ui-monospace, monospace;
  --display-weight:   500;
  --display-tracking: -0.02em;
}
```

SCSS variable rebinding in `_variables.scss`:

```scss
$brand_color:       #5BE0B0;  // --primary
$brand_text_color:  #0B2A22;  // --primary-ink
$text_color:        #F2F4F9;  // --ink
$background_color:  #0C172F;  // --bg
$link_color:        #5BE0B0;  // --primary
$hover_link_color:  #2BB089;  // --brand-mint-deep
$visited_link_color:#6FA3E8;  // --accent
$button-color:      #5BE0B0;
$secondary-text-color: #B6C3DD;
```

## Page-by-page spec

### Header (`templates/header.hbs`)

Markup matches Coverly's structure with Zendesk helpers wired in:

- `.brand` → `{{#link 'help_center'}}` with `{{asset 'coverly-icon.png'}}` (or `settings.logo` if set) + `{{help_center.name}}` + " / Help Center" sub
- `.nav` (desktop): `{{link 'community'}}`, `{{link 'service_catalog'}}`, **static status chip** ("All systems normal"), `{{#link 'new_request' class='pill'}}` (Contact pill)
- When `signed_in`: user dropdown with avatar + name, opening to profile / requests / contributions / approvals / contact details / change password / sign out
- When not signed in: `{{#link 'sign_in'}}` link
- Mobile drawer (`.nav-wrapper-mobile`): hamburger button + `<nav class="menu-list-mobile">` containing all of the above, restyled for dark surface
- Skip-navigation link kept (`<a class="skip-navigation">`)
- **"Ask AI" link removed**

### Footer (`templates/footer.hbs`)

- 4-column `.footer-grid`:
  - **Brand column**: logo + name + signature paragraph (static copy — "Travel eSIMs that just work…")
  - **Help column**: `help_center`, `getting-started` category placeholder (admin can edit), `community`, `new_request`
  - **Coverly column**: 4 static marketing links → `#` placeholders ("About us", "Press", "Careers", "Affiliates")
  - **Legal column**: 4 static links → `#` placeholders ("Terms of service", "Privacy policy", "Fair-use policy", "System status")
- `.footer-bottom`: copyright + static status chip + language selector dropdown (using existing Copenhagen `alternative_locales` markup, restyled)

### `document_head.hbs`

Add inside `<head>`:
```hbs
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Newsreader:ital,opsz,wght@0,6..72,400;0,6..72,500;0,6..72,600;1,6..72,400&family=Manrope:wght@400;500;600;700&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet">
```
Keep the existing import-map and flash-notifications bootstrap unchanged.

### Home page (`templates/home_page.hbs`)

Sections top-to-bottom:

1. **`.hero[data-style="photo"]`** with background-image `{{asset 'umbrellas.jpg'}}` (or `settings.homepage_background_image` override):
   - Static `.hero-eyebrow`: "Coverly Help Center · 190+ countries"
   - Display `<h1>` with italic accent (static copy)
   - Static `.lead` paragraph
   - Search: `{{search submit=false instant=settings.instant_search class='searchbar'}}` wrapped in `.search-glow` for the animated conic-gradient
   - `.quick-chips`: 5 static popular-search links (admin can edit template to change targets)
   - `.hero-stats`: 3 static stats (190+, ~38s, 4.8/5)

2. **`.status-banner`**: static dot + headline + lead + linkish ("View status page" → `#` placeholder). Pulls up over hero seam via negative margin.

3. **`.cat-grid`** (3-col): `{{#each categories}}` rendering `.cat-card`. Each card:
   - `.icon` with inline-SVG picked by `{{#is id 'getting-started'}}…{{/is}}` chain mapping known category slugs to SVGs (compass / download / globe / wallet / wrench / suitcase). Default fallback SVG when no match.
   - `<h3>{{name}}</h3>`
   - `<p>{{excerpt description}}</p>`
   - `.meta`: "{{sections.length}} sections" + arrow

4. **`.trending-grid`** (2-col, 6 cells): `{{#if promoted_articles}} {{#each promoted_articles}}` rendering `.trending-item`. Each item:
   - Numbered eyebrow rendered via **CSS counters** (no template arithmetic): `.trending-grid { counter-reset: trending; }` and `.trending-item::before { counter-increment: trending; content: counter(trending, decimal-leading-zero); }`
   - `<h4>{{title}}</h4>`
   - `.meta`: static "Promoted article" eyebrow (Zendesk `promoted_articles` doesn't include section/category context)
   - Click → `{{url}}`
   - The `{{#if promoted_articles}}` guard hides the section if nothing is promoted

5. **Contact CTA block** (`.cta-block`):
   - Left: eyebrow "Still stuck?" + display `<h2>` + lead + `{{#link 'new_request' class='submit-btn'}}` "Contact support"
   - Right: two `.contact-card`s (live chat + email — static copy)

### Category page (`templates/category_page.hbs`)

```
.wrap
  [breadcrumbs] {{breadcrumbs}}
  .page-header
    .eyebrow.mono "Category"
    <h1>{{category.name}}</h1>
    <p class="lead">{{category.description}}</p>
    .stats — "{{sections.length}} sections · Updated this week"
    (Coverly's design also shows a total article count; we omit it because Curlybars has no sum helper and the per-section count below carries the same information.)
  {{#each sections}}
    .section-block
      <h3>{{name}}</h3>
      .sec-meta — link to section page + " · {{articles.length}} articles"
      .article-list
        {{#each articles}}
          <a class="article-row" href="{{url}}">
            <div>
              <div class="article-title">{{title}}</div>
              <div class="article-sub"><clock-icon> Updated {{date edited_at timeago=true}}</div>
            </div>
            {{#if content_tags}}
              <div class="badges">{{#each content_tags}}<span class="tag">{{name}}</span>{{/each}}</div>
            {{/if}}
          </a>
        {{/each}}
      {{#if more_articles}}<a class="see-all-articles" href="{{url}}">{{t 'show_all_articles' count=article_count}}</a>{{/if}}
  {{/each}}
```

Total article count is computed inline in the template if Curlybars supports it; otherwise, fall back to omitting that stat element.

### Section page (`templates/section_page.hbs`)

```
.wrap
  [breadcrumbs]
  .page-header
    .eyebrow.mono "Section"
    <h1>{{section.name}}</h1>
    <p class="lead">{{section.description}}</p>
    .stats — "{{articles.length}} articles · In {{category.name}}"
    {{#if settings.show_follow_section}}{{subscribe}}{{/if}}
  .section-block
    .article-list — same article-row markup as category page
  {{pagination "section.articles"}}
```

Nested-sections handling preserved (Copenhagen renders `section.sections` for hierarchical KBs) but with article-row styling.

### Article page (`templates/article_page.hbs`)

```
.wrap-narrow
  [breadcrumbs]
  .article-page
    {{#if settings.show_articles_in_section}}
      <aside class="article-sidebar">
        — Copenhagen's existing collapsible sidebar markup, restyled to use --bg-elev / --line tokens
      </aside>
    {{/if}}
    .article-header
      .eyebrow.mono "{{section.category.name}} · {{section.name}}"
      <h1 class="display">{{article.title}}</h1>
      .article-meta:
        {{#if settings.show_article_author}}.author with {{user_avatar}} + {{article.author.name}}{{/if}}
        .meta-item "Updated {{date article.edited_at timeago=true}}"
        {{#if article.content_tags}}.tag chips, right-aligned{{/if}}
        {{#if settings.show_follow_article}}{{subscribe}}{{/if}}
    <article class="article-body">{{article.body}}</article>
    {{#if attachments}}
      .article-attachments — styled with --bg-elev / --line
    {{/if}}
    .helpful
      <h4>{{t 'was_this_article_helpful'}}</h4>
      <div class="helpful-buttons">
        {{vote 'up' class='button article-vote' selected_class='selected'}}
        {{vote 'down' class='button article-vote' selected_class='selected'}}
      </div>
      <small>{{vote 'label'}}</small>
    {{#if settings.show_article_sharing}}{{share}}{{/if}}
    {{#if settings.show_recently_viewed_articles}}{{recent_articles}}{{/if}}
    {{#if settings.show_related_articles}}
      <section class="related">
        .section-head — eyebrow "Related" + h2 "More from {{section.name}}"
        .related-grid
        {{related_articles}}
      </section>
    {{/if}}
```

**Article-body styling** (`_article.scss`):
- Style standard WYSIWYG elements: `h2`, `h3`, `h4`, `p`, `ul`, `ol`, `li`, `blockquote`, `a`, `code`, `pre`, `img`, `figure`, `table`
- Style opt-in classes: `.callout` (left-accent block), `.step-card` (numbered card). Authors paste raw HTML with these classes to get the styled treatment. Document the convention in `AGENTS.md`.

**Dropped**:
- Custom feedback textarea (Coverly's "What were you looking for that we missed?" — Zendesk's vote helper doesn't have a freeform comment channel)
- Article comments block (per your decision)
- Return-to-top inline link

## `manifest.json`

Kept settings:
- `logo` (file)
- `show_brand_name` (checkbox)
- `favicon` (file)
- `homepage_background_image` (file)
- `instant_search` (checkbox)
- `scoped_kb_search` (checkbox)
- `show_articles_in_section` (checkbox)
- `show_article_author` (checkbox)
- `show_follow_article` (checkbox)
- `show_follow_section` (checkbox)
- `show_recently_viewed_articles` (checkbox)
- `show_related_articles` (checkbox)
- `show_article_sharing` (checkbox)
- `show_suggested_articles` (checkbox) — consumed by new-request-form React module

Removed settings:
- `brand_color`, `brand_text_color`, `text_color`, `link_color`, `hover_link_color`, `visited_link_color`, `background_color`
- `heading_font`, `text_font`
- `show_article_comments`, `show_recent_activity`, `scoped_community_search`
- `community_background_image`, `community_image`, `service_catalog_hero_image`

Theme `name` stays `"Coverly"`; `version`, `author`, `api_version`, `default_locale` unchanged.

`translations.yml` — strip out keys for removed settings.

## Data wiring & template helpers

| UI element | Source |
|---|---|
| Logo | `{{asset 'coverly-icon.png'}}` (default), `{{settings.logo}}` (override) |
| Hero background | `{{asset 'umbrellas.jpg'}}` (default), `{{settings.homepage_background_image}}` (override) |
| Hero eyebrow, h1, lead, stats, quick-chips | Hardcoded in template |
| Hero search | `{{search submit=false instant=settings.instant_search}}` |
| Status banner | Hardcoded |
| Categories grid | `{{#each categories}}` — name, description, sections.length, url |
| Category icons | Inline-SVG dictionary keyed by category slug; default fallback SVG |
| Trending grid | `{{#each promoted_articles}}` — title, url, optional internal-lock icon |
| Contact CTA | Static copy + `{{#link 'new_request'}}` |
| Breadcrumbs | `{{breadcrumbs}}` |
| Article author | `{{article.author.name}}`, `{{user_avatar}}` (gated by setting) |
| Article body | `{{article.body}}` (WYSIWYG-emitted HTML) |
| Article content tags | `{{#each article.content_tags}}` |
| Helpful vote | `{{vote 'up'}}`, `{{vote 'down'}}`, `{{vote 'label'}}` |
| Related articles | `{{related_articles}}` (gated by setting) |
| Recently viewed | `{{recent_articles}}` (gated by setting) |
| Subscribe | `{{subscribe}}` (gated by settings) |
| Sharing | `{{share}}` (gated by setting) |
| Footer language | `{{#each alternative_locales}}` |

## Risks & known gaps

1. **Animated search glow** (`--ang` `@property` conic-gradient) is unsupported on older Firefox/Safari. The CSS already has a `@supports not` fallback that hides the gradient. Acceptable.
2. **Category icons hardcoded by slug.** New categories whose slugs aren't in the dictionary get a default icon. Document the convention in `AGENTS.md` so admins know they can add SVGs.
3. **`{{search instant=true}}` suggestions UI** is rendered by Zendesk, not us. We can style the dropdown container but it won't perfectly match Coverly's bespoke suggestion-row design. The input and submit button will match.
4. **`promoted_articles` is the only data source for trending.** If fewer than 6 articles are promoted, the trending grid has fewer cells. Admin should promote ~6 for the layout to feel complete. Documented.
5. **Out-of-scope pages may have contrast issues** on the new dark palette (Copenhagen partials weren't dark-mode-designed). We patch the worst offenders in their existing partials (e.g. ensure form inputs are readable) but a full audit is out of scope.
6. **No automated tests.** Verification is manual via local preview.

## Testing approach

- `yarn build` succeeds, no Rollup or SCSS errors.
- `yarn eslint src` passes (no source-code changes expected, but the eslint config flags shared module imports — confirm we haven't touched anything that breaks it).
- `zcli themes:preview` launches; visual check of:
  - Home page (hero, search, status banner, categories grid, trending grid if promoted articles exist, contact CTA, footer)
  - One category page
  - One section page within that category
  - One article page (with and without `show_articles_in_section` enabled)
  - One out-of-scope page (e.g., a request form) — confirm header/footer render correctly and palette inheritance hasn't broken inputs/buttons/typography
- Firefox only (per `AGENTS.md` constraint — Chromium has a CORS bug in local preview).
- Login is interactive; the implementing agent pauses and asks the user to authenticate at preview time.

## Open questions

None at the time of writing. All scope, architecture, and data-wiring decisions are documented above.
