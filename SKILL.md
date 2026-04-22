---
name: ed-shopify-theme-rules
description: "Dawn theme CSS pitfalls, Liquid/section schema gotchas, and static-to-theme migration workflow. Trigger on SYMPTOMS of known Dawn pitfalls: empty div not showing, CSS被覆盖/!important不生效, rem值太小, section不显示/preset rejected, cart drawer报错/error updating cart, backdrop-filter导致fixed偏移, gradient被覆盖, html height异常, button hover不生效, 响应式断点错位(750px/990px), z-index冲突导致菜单被遮挡, CDN缓存导致修改不可见, filter不工作, price filter not filtering, 价格筛选不生效, sticky不生效, URL格式错误/链接不生效. Also trigger for: Dawn CSS conflict, Dawn base.css override, section schema validation error, Liquid template bug, settings_schema.json, image_picker, shopify theme push/pull, static-to-Shopify conversion, url type setting format, shopify:// resource URL, template JSON link format, collection filter, price range filter, Search and Discovery app. Chinese: Dawn样式冲突, section不显示, 样式不生效, 主题推送, 静态转主题, 筛选器不工作. NOT for API/backend (use ed-shopify-api-rules), NOT for curl mirroring (use ed-website-cloning-rules)."
---

# Shopify Theme Development Rules

Pitfalls and best practices for Shopify theme development, especially Dawn theme customization.

## Dawn Theme

- **Dawn theme overrides**: See `DAWN_THEME_GUIDE.md` (if present in project) for comprehensive Dawn-specific CSS specificity traps, schema gotchas, migration checklists, and debugging patterns.

## Dawn CSS Pitfalls

- **`*:empty { display: none }` kills overlay/spacer/link divs.** Dawn's `base.css` (~line 481) hides ALL empty elements: `a:empty, div:empty, section:empty, p:empty, h1-h6:empty { display: none; }`. This affects overlays, spacer divs, full-area clickable `<a>` tags, `::before`/`::after` parent elements — anything with no child text nodes. **Fix**: add `&nbsp;` inside the element + `font-size:0; line-height:0;` in CSS. **This is the #1 most common Dawn pitfall** — check EVERY empty element you create.
- **`::before`/`::after` pseudo-elements don't count as children.** A `<div>` with only `::before` content is still `:empty` in CSS. The pseudo-element won't save it from `display: none`. You must add actual DOM content (`&nbsp;`).
- **Inline `<style>` in theme.liquid beats external CSS.** Dawn generates color scheme CSS variables in an inline `<style>` tag inside `theme.liquid`. External CSS files loaded via `stylesheet_tag` have lower cascade priority. **Fix**: modify `config/settings_data.json` color schemes directly instead of trying to override with external CSS.
- **Dawn `.grid` is `display: flex`, NOT CSS Grid.** Dawn's `.grid` class uses `display: flex; flex-wrap: wrap;` with child widths calculated via `width: calc(50% - var(--grid-desktop-horizontal-spacing) / 2)`. The `.product.grid` has `gap: 0` and uses padding for spacing. **Never add `gap` to `.product.grid`** — it breaks the width calculations and causes flex items to wrap to the next line.
- **Dawn section CSS loads AFTER your theme CSS.** Each section loads its own CSS inline via `{{ 'section-main-product.css' | asset_url | stylesheet_tag }}` inside the section Liquid. This means Dawn's component CSS has higher cascade priority than custom CSS in `<head>`. **Fix**: always use `!important` when overriding Dawn's `display`, `flex`, `width`, `max-width`, or `position` rules.
- **Dawn custom elements create deep nesting.** `<product-form>`, `<media-gallery>`, `<quantity-input>`, `<slider-component>` are custom HTML elements that add invisible DOM layers. A "simple" Add to Cart button is actually nested 4 levels deep: `product-form > form > .product-form__buttons > button`. **Always inspect real DOM with DevTools before writing CSS selectors** — the Liquid source gives a misleading mental model.
- **`display: contents` on `<form>` is unreliable.** Using `display: contents` to "flatten" nested elements for CSS Grid/Flex doesn't work reliably on `<form>` elements, especially with Shopify's dynamic payment button (`{{ form | payment_button }}`). **Fix**: when a deeply nested element needs to break out of its parent's layout, use a small JS script (`element.after(child)`) to move it in the DOM instead.
- **Dawn `product-form__input` has hidden flex rules.** `.product-form__input { flex: 0 0 100%; max-width: 44rem; min-width: fit-content; }` — these are dormant in block flow but **activate immediately** when you set `display: flex` on the parent container, causing all inputs to force full width. Override with `flex: 0 0 auto !important; width: auto !important; max-width: none !important;`.
- **Dawn `html { display: grid }` breaks full-height layouts.** Dawn's `theme.liquid` inline style sets `html { display: grid; min-height: 100%; }` and `body { display: grid; grid-template-rows: auto auto 1fr auto; }`. This causes: (1) custom sections with `height: 100vh` to behave differently than in static HTML, (2) `body > *` elements to stretch full width unexpectedly, (3) absolute/fixed positioned elements to miscalculate their containing block. **Fix**: if your custom section needs predictable height, use explicit `px` or `vh` values on the section itself, not on html/body. Don't override Dawn's html/body grid — it controls header/footer/content flow.
- **Dawn `base.css` global button/link styles silently override custom hover effects.** `base.css` sets `button { font-size: 1.5rem; letter-spacing: 0.1rem; }`, `a { color: rgb(var(--color-foreground)); }`, and `.button:hover { box-shadow: ... }`. These fire on ALL buttons and links, including ones inside custom sections. **Symptoms**: hover text color doesn't change, font-size on custom buttons is wrong, unexpected box-shadow appears. **Fix**: always override with full specificity on your custom class — `.your-section .your-button:hover { color: #fff !important; box-shadow: none !important; }`. Never rely on just `.your-button` — Dawn's tag-level selectors tie in specificity and win by cascade order.
- **Dawn gradient/background-color overrides are invisible in DevTools at first glance.** Dawn's color scheme system generates `background: rgb(var(--color-background));` on `.color-scheme-1` (and 2, 3, etc.) which wraps every section. Your custom `background: linear-gradient(...)` on a child element appears to work in DevTools Styles panel, but the parent's solid background bleeds through transparent gradient stops. **Fix**: set `background: transparent !important;` on the `.color-scheme-*` wrapper of your section, or use the schema `"class"` field to opt out of color schemes entirely.

## Dawn Responsive Breakpoints (NOT standard breakpoints)

- **Dawn uses `750px` and `990px`, NOT 768px and 1024px.** Dawn's media queries are `@media screen and (min-width: 750px)` for tablet and `@media screen and (min-width: 990px)` for desktop. If you use 768px/1024px in custom CSS, there's a gap where Dawn shows mobile layout but your CSS shows tablet layout (750-768px) or Dawn shows tablet but your CSS shows desktop (990-1024px). **Fix**: always use `750px` and `990px` in custom section media queries to match Dawn. Additional breakpoints: `1200px` (large desktop, used by `.page-width` max-width) and `450px` (small mobile, used by some Dawn components).
- **Dawn's responsive class convention**: `.small-hide` = hidden below 750px, `.medium-hide` = hidden below 990px, `.large-up-hide` = hidden above 990px. Use these utility classes in Liquid instead of writing duplicate media queries.

## Dawn CSS Selector Traps (from real debugging sessions)

- **Dawn container class names differ from what you'd guess.** The collection grid wrapper is `.facets-vertical.page-width`, NOT `.collection.page-width`. Filter headings use `.facets__summary .facets__summary-label`, NOT `.facets__heading`. Price filter title has no `.facets__summary-label` at all — it's a bare `<span>`. **Always inspect live DOM before writing selectors.**
- **Dawn `.button-label` overrides your font-size.** `.button-label` in `base.css` sets `font-size: 1.5rem` globally. If you target `.mobile-facets__open` for font-size, `.button-label` wins. **Fix**: use compound selector `.mobile-facets__open .mobile-facets__open-label.button-label` to beat specificity.
- **Dawn caret icon class is `.icon-caret`, not `.svg-wrapper`.** The inline SVG caret in filter accordions uses class `.icon-caret`. Targeting `.svg-wrapper` to hide it does nothing.
- **Dawn `<details>` open state cannot be controlled via CSS.** To make a specific filter group open by default, you must modify `facets.liquid` Liquid logic (e.g. `{% if filter.param_name contains 'tag' %}open{% endif %}`). CSS cannot add the `open` attribute.
- **Dawn multi-layer containers cause padding stacking.** `.page-width` has `padding: 0 1.5rem` (mobile) / `0 5rem` (desktop). If you add padding to child containers (`.product-grid-container`, `.collection`), they **stack**. **Fix**: set padding on outermost container only, zero out inner layers.
- **Dawn `enable_sorting: false` can crash vertical filter layout.** Setting `enable_sorting` to `false` in template JSON while `filter_type` is `vertical` may break the collection template entirely. **Fix**: keep `enable_sorting: true` in JSON, use CSS to hide sort elements instead (`display: none !important`).
- **Dawn "Filter and sort" vs "Filter" text is logic-controlled.** Mobile shows `filter_and_sort` translation when both filtering and sorting are enabled. Can't change via CSS text replacement. **Fix**: Dawn renders two `<span>` labels — one for mobile (`medium-hide large-up-hide`) showing "Filter and sort", one for tablet+ (`small-hide`) showing "Filter". Use CSS to hide the mobile label and force-show the tablet label on all sizes.
- **Dawn grid spacing uses CSS custom properties from `settings_data.json`.** `--grid-desktop-horizontal-spacing` and `--grid-desktop-vertical-spacing` are computed from `spacing_grid_horizontal` / `spacing_grid_vertical` settings. Mobile values are auto-halved. Override these variables on `#product-grid.product-grid` to change spacing.
- **Dawn card ratio uses `--ratio-percent` inline style.** The `card--standard.ratio` element has `style="--ratio-percent: 125%"` set inline. Override with `.collection .card.card--standard.ratio { --ratio: 0.75 !important; }` for 3:4 portrait.

## Horizontal scroll containers need overflow-y: hidden

- **`overflow-x: auto` without `overflow-y: hidden` causes scroll trapping.** On touch devices and trackpads, the browser captures vertical scroll events inside the horizontal scroll container, making the page feel "stuck". Users can't scroll past the section. **Fix**: always pair `overflow-x: auto` with `overflow-y: hidden` on carousel tracks and horizontal scroll containers.

## Dawn font inheritance overrides custom fonts

- **Dawn's body inherits `font-family: Inter`, `line-height: 1.8`, and `letter-spacing: 0.06rem` to ALL elements.** Even if you load Google Fonts and set CSS custom properties, only elements with explicit overrides use them. Everything else inherits Dawn's values. `line-height: 1.8` is the #1 silent height killer — it makes every section ~20% taller than the static HTML reference. **Fix**: force all three properties on ALL custom section elements: `[class^="home-"] { font-family: var(--home-font-body) !important; line-height: 1.5 !important; letter-spacing: 0 !important; }`. Also force heading font + line-height separately.

## Dawn font inheritance broken by tag-level selectors (set color = must set font-family)

- **Every CSS selector that sets `color` MUST also set `font-family` and `font-weight` explicitly.** Dawn's `base.css` and `theme.liquid` set `font-family` on tag-level selectors (`body`, `h1-h6`, `a`, `button`, `strong`, `p`). These tag-level rules have equal or higher specificity than class-based inheritance. A child element that only sets `color` (without `font-family`) silently falls back to Dawn's default font (e.g. Assistant/Inter). The visual inconsistency is amplified by color differences — a darker element in the wrong font looks dramatically different from its lighter sibling, even though the bug is font, not color. **This is the #1 cause of "two labels side-by-side look like different fonts".**

- **Two-layer defense (both required):**

  **Layer 1 — Hijack Dawn's CSS variables at source** in your base CSS file loaded from `theme.liquid`:
  ```css
  :root {
    --font-body-family: "Your Font", fallback, serif !important;
    --font-heading-family: "Your Font", fallback, serif !important;
  }
  ```
  This makes Dawn's own components (product grid, facets, cart drawer, pagination, mobile menu) use your font automatically — no need to override each Dawn component individually.

  **Layer 2 — Explicit font on every color-setting selector.** When writing section CSS, apply this checklist to EVERY selector:
  - Sets `color`? → MUST add `font-family` + `font-weight`
  - Is a `<strong>`, `<a>`, `<button>`, or `<p>` inside a custom section? → MUST add `font-family` (Dawn has global rules on these tags)
  - Is a state modifier (`.is-active`, `.is-secondary`, `--outline`)? → MUST add `font-family` (modifiers can lose the base selector's font in cascade)
  - Is a child `<span>` with different color than parent? → MUST add `font-family`

- **Audit command — run after building all sections to find violations:**
  ```bash
  grep -n "color:" sections/your-prefix-*.liquid | grep -v "font-family" | grep -v "background\|border\|box-shadow\|text-shadow\|outline\|caret\|accent\|stroke\|fill\|/\*"
  ```
  Any result is a potential font inheritance bug. Verify each one and add explicit `font-family`.

- **Why Layer 1 alone is NOT sufficient:** `--font-body-family` only affects elements that use `var(--font-body-family)`. Dawn's `base.css` also sets fonts via direct `font-family` declarations on `button`, `.button-label`, `input`, `select`, etc. These don't reference the variable. Layer 2 catches these cases.

## Dawn checkbox/filter positioning breaks after font change (verified 2026-04)

- **Dawn's checkbox and filter components use hardcoded `rem` values for absolute positioning, calibrated to the default font (Assistant/Inter).** When you change `--font-body-family` globally, the line-height and text metrics change, causing checkmarks and checkbox inputs to misalign. **Symptoms**: checkmark appears below/outside the checkbox square, checkbox overlaps with text, filter drawer labels misaligned.

- **Fix Dawn's CSS directly, don't override.** Edit `assets/component-facets.css` and replace all hardcoded position values with font-agnostic centering:
  ```css
  /* BEFORE (breaks with custom fonts): */
  .mobile-facets__label .icon-checkmark { top: 1.9rem; left: 2.8rem; }
  input.mobile-facets__checkbox { top: 1.2rem; left: 2.1rem; }
  .facet-checkbox .svg-wrapper { top: 1.4rem; }

  /* AFTER (works with any font): */
  .mobile-facets__label .icon-checkmark { top: 50%; transform: translateY(-50%); left: 2.8rem; }
  input.mobile-facets__checkbox { top: 50%; transform: translateY(-50%); left: 2.5rem; }
  .facet-checkbox .svg-wrapper { top: 50%; transform: translateY(-50%); }
  ```
  Also add `align-items: center` to `.mobile-facets__label` (Dawn only sets `display: flex` without it).

- **Never use CSS wildcards (`*`) on Dawn component containers.** Rules like `.facets-vertical.page-width * { line-height:1.5!important }` break checkboxes, radio buttons, SVG icons, and `<input>` elements inside Dawn's facets. Dawn's checkbox relies on default `line-height` for positioning — a wildcard override destroys the layout.

- **`.facets__label` is shared between desktop sidebar AND mobile filter drawer.** The mobile `<label>` has BOTH classes: `class="facets__label mobile-facets__label"`. If you style `.facets__label { padding:6px 0!important }` for the desktop sidebar, it also kills mobile drawer padding. **Fix**: scope with `.facets-vertical .facets__label:not(.mobile-facets__label)` for desktop-only styles.

- **Order of operations when changing global fonts in Dawn:**
  1. Override `:root` CSS variables (`--font-body-family`, `--font-heading-family`)
  2. Fix `component-facets.css` checkbox positioning → `top:50%; transform:translateY(-50%)`
  3. In custom section CSS, NEVER target `.facets__label` / `.mobile-facets__label` / `.mobile-facets__checkbox` — let Dawn handle its own components
  4. If you must style desktop filter labels, always exclude mobile: `:not(.mobile-facets__label)`

## Dawn default `<p>`/`<h>` margins inflate section height

- **Dawn's `base.css` adds `margin-block-start: 1em` to `<p>` and `<h1-h6>` elements.** Even after fixing font/line-height/letter-spacing, sections are still taller than the static HTML reference because every `<p>` and heading gets ~14-18px extra `margin-top`. If a section has many `<p>` tags (e.g. 4 per tab × 3 tabs = 12 elements), the accumulated extra margins can add **100+ px** of height. **Fix**: add a global margin-top reset in your base CSS — but **only reset `margin-top`**, not `margin-bottom`, to avoid killing intentional spacing set by component CSS:
  ```css
  [class^="home-"] p, [class^="home-"] h1, [class^="home-"] h2, [class^="home-"] h3,
  [class*=" home-"] p, [class*=" home-"] h1, [class*=" home-"] h2, [class*=" home-"] h3 {
    margin-top: 0 !important;
  }
  ```
  **Do NOT use `margin: 0 !important`** — this kills component-level `margin-bottom` values, collapsing all spacing between elements.

## Pixel-perfect debugging: Console dump comparison

- **When a section's height doesn't match the static HTML, use a recursive Console script to dump every element's computed margin/padding/line-height on both pages, then diff line by line.** Visual inspection and clicking through DevTools is too slow for multi-layer nesting. The script walks every child of a container and logs `h`, `mt`, `mb`, `pt`, `pb`, `fs`, `lh` per element. Run on both Shopify and the static HTML, then compare side by side. This approach found 6 micro-differences (padding, line-height, font-size, margin-bottom, unwanted p margins) that added up to exactly the height mismatch. **Always confirm class selectors exist on both pages** — Shopify sections use BEM classes while the static HTML may use Tailwind utilities, requiring a fallback selector in the script.

**Full comparison script (run on both pages, diff outputs):**
```js
(function(sel){var r=document.querySelector(sel);if(!r)return console.log('NOT FOUND:',sel);
function dump(el,d){var s=getComputedStyle(el),t=el.tagName+'.'+[...el.classList].join('.');
console.log(' '.repeat(d)+t,'h:'+el.offsetHeight,'mt:'+s.marginTop,'mb:'+s.marginBottom,
'pt:'+s.paddingTop,'pb:'+s.paddingBottom,'fs:'+s.fontSize,'lh:'+s.lineHeight,
'ff:'+s.fontFamily.split(',')[0],'ls:'+s.letterSpacing,'fw:'+s.fontWeight);
[...el.children].forEach(c=>dump(c,d+1));}dump(r,0);})('.your-section-class');
```
This covers font-family, letter-spacing, and font-weight in addition to spacing — the three properties Dawn most commonly overrides silently.

## Dawn rem ≠ standard rem

- **Dawn uses `font-size: 62.5%` on `<html>`, making `1rem = 10px` (not 16px).** This means `0.875rem` gives you 8.75px, not 14px. If your CSS was authored for standard 16px base (e.g. from Tailwind/static HTML conversion), ALL `rem` values will be ~37% smaller than intended. **Fix**: use `px` exclusively in custom section CSS, or multiply rem values by 1.6. This is the most common visual mismatch when converting static HTML to Dawn themes — fonts and spacing look too small.

## Dawn position: sticky vs fixed

- **`position: sticky` header won't reappear on scroll-up.** Sticky elements only stick within their parent container. Once the parent (Dawn's `.shopify-section` wrapper) scrolls out of viewport, the header can't re-stick. **Fix**: use `position: fixed` instead, and add `padding-top` to the wrapper to compensate for the removed-from-flow header.
- **Fixed header + announcement bar requires coordinated z-index.** Both need `position: fixed` with announcement bar above header (`z-index: 1001` vs `1000`). Header needs `top: 3.6rem` (announcement height). The Dawn section wrapper (`.wz-header-wrapper`) also needs `z-index: 1000` to establish a stacking context that beats other sections, otherwise mega menus get clipped.
- **`backdrop-filter` on a parent breaks `position: fixed` children.** CSS spec: `filter`, `backdrop-filter`, `transform`, `perspective`, or `contain` on an element makes it a **new containing block** for all `position: fixed` descendants. This means a fixed-position mega menu inside a header with `backdrop-filter: blur(10px)` will be positioned relative to the header, NOT the viewport. The mega menu's `top` value becomes offset by the header's own position, creating a visible gap. On hero pages this is invisible (dark hero fills the gap); on non-hero pages the gap exposes page content. **Fix**: move `backdrop-filter` to a `::before` pseudo-element on the header — the pseudo creates its own stacking context without affecting the parent's containing block behavior:
  ```css
  .header--scrolled::before {
    content: ''; position: absolute; inset: 0;
    backdrop-filter: blur(10px); -webkit-backdrop-filter: blur(10px);
    z-index: -1;
  }
  ```
  This preserves the blur effect while keeping `position: fixed` children relative to the viewport.

## `position: fixed` elements ignore parent stacking context for z-index

- **A `position: fixed` element competes at the viewport level, NOT within its parent's stacking context.** Setting `z-index: 1; position: relative` on a section wrapper does NOT constrain a `position: fixed` child's z-index. The fixed child competes directly with other fixed/positioned elements (like a mobile menu drawer) at the viewport root. This means a section with `position: fixed; z-index: 10` will overlap a mobile menu with `position: fixed; z-index: 110` only if 110 > 10 — but if any section element has z-index ≥ the menu's z-index, it will float on top of the menu, blocking clicks.
- **Symptoms**: mobile menu opens but page content (text, toolbars, sticky headers) shows through and blocks menu interaction. Looks like a transparency issue but is actually a z-index stacking issue.
- **Common offenders in Dawn themes**: `position: sticky` toolbars with `z-index: 50`, hero section content with `z-index: 10`, scroll-driven sections using `position: fixed` for sticky polyfill.
- **Fix**: audit ALL `z-index` values across all sections and CSS files in the theme. Every non-overlay element should use `z-index ≤ 5`. Reserve `z-index: 100+` exclusively for overlays (mobile menu, cart drawer, modals). Don't try to fix this per-page — do a **global search for `z-index`** and lower everything in one pass.
- **Diagnostic**: open mobile menu, then run `document.elementFromPoint(x, y)` at the blocked area. Walk up the ancestor chain logging `position` and `zIndex` to find the offending element.
- **Critical rule**: when creating any `position: fixed` or `position: sticky` element in a custom section, always set its `z-index` below the mobile menu's z-index (check `clay-header.css` or equivalent for the menu's value). Never use `z-index: 10+` on section content — it WILL eventually conflict with navigation overlays.

## Custom Section CSS Isolation (Critical for Static-to-Theme Migration)

When building custom sections with a CSS namespace (e.g. `.os-`, `.wz-`), you MUST follow these rules:

### Never use wildcard resets inside wrapper selectors

**WRONG — will break everything:**
```css
.os-wrapper *, .os-wrapper *::before, .os-wrapper *::after {
  margin: 0; padding: 0; box-sizing: border-box;
  font-family: inherit; color: inherit;
}
```

**Why it breaks:** Shopify loads CSS per-section via `{{ 'file.css' | asset_url | stylesheet_tag }}`. If multiple sections load the same base CSS file, it appears multiple times in the DOM. The wildcard reset from the LAST `<link>` has equal specificity to your specific class rules but comes later in cascade order — so it wins, zeroing out all padding/margin on every element. This is the **#1 cause of "all my spacing disappeared"** bugs.

**RIGHT — only reset what Dawn forces, nothing else:**
```css
.os-wrapper *, .os-wrapper *::before, .os-wrapper *::after {
  box-sizing: border-box;
  letter-spacing: normal;  /* Override Dawn body's 0.06rem */
}
```

### Dawn's three layers of style interference

Dawn applies styles at three levels. You must override each precisely:

**Layer 1: `html` (theme.liquid inline style)**
```css
html { font-size: calc(var(--font-body-scale) * 62.5%); } /* 1rem = 10px */
```
→ Use `px` exclusively in all custom CSS. Never use `rem`.

**Layer 2: `body` (theme.liquid inline style)**
```css
body {
  font-size: 1.5rem;          /* = 15px, inherited by ALL children */
  letter-spacing: 0.06rem;    /* = 0.6px, inherited by ALL children */
  line-height: calc(1 + 0.8 / var(--font-body-scale));  /* ~1.8 */
  font-family: var(--font-body-family);  /* e.g. Assistant, sans-serif */
  display: grid;               /* affects section layout */
}
```
→ On your wrapper: `font-size: 16px !important; line-height: 1.5 !important; letter-spacing: normal !important;`
→ Do NOT set `color` on the wrapper if children need different colors.

**Layer 3: `h1-h6` (base.css)**
```css
h1, h2, h3, h4, h5 {
  font-family: var(--font-heading-family);  /* e.g. Assistant */
  font-weight: var(--font-heading-weight);  /* often 400, NOT 700! */
  color: rgb(var(--color-foreground));       /* often dark gray */
  line-height: calc(1 + 0.3 / max(1, var(--font-heading-scale)));
  letter-spacing: calc(var(--font-heading-scale) * 0.06rem);
  word-break: break-word;
}
```
→ Override globally: `.wrapper h1-h6 { font-weight: 700 !important; letter-spacing: normal !important; word-break: normal !important; font-style: normal !important; }`
→ Do NOT override `font-family` or `color` globally — let each specific class set its own with `!important`.

### Per-element `!important` is mandatory

Every custom class that sets `color`, `font-family`, or `line-height` must use `!important`, because Dawn's h1-h6/body rules target tag names (higher cascade origin than class selectors in some contexts):
```css
.os-hero-heading {
  font-family: 'Montserrat', Georgia, serif !important;
  color: #0d2e25 !important;
  line-height: 1.1 !important;
}
```

### Section wrapper sticky/z-index for card-stack effects

Shopify wraps each section in `<div class="shopify-section">`. For card-stack or parallax-over effects:
- Use the schema `"class"` field to add your own class to the wrapper: `"class": "os-hero-wrapper"`
- Then target: `.os-hero-wrapper { position: sticky; top: 0; z-index: 1; }`
- The next section: `.os-about-wrapper { position: relative; z-index: 5; }` — slides over the hero

### CSS debugging: compare computed values, don't guess

When a visual mismatch exists between static HTML and Shopify theme, run this in Console on BOTH pages:
```js
(function(){var el=document.querySelector('.your-class');if(!el)return;var s=getComputedStyle(el);
console.table({fontSize:s.fontSize,fontWeight:s.fontWeight,fontFamily:s.fontFamily,
color:s.color,padding:s.padding,lineHeight:s.lineHeight,letterSpacing:s.letterSpacing})})()
```
Compare the two outputs side by side. The differing property IS the bug. Do this BEFORE touching CSS — never guess at specificity issues.

## Dawn CSS Override Workflow

When overriding Dawn's built-in sections (main-product, header, etc.):

1. **DevTools first** — inspect the live DOM, screenshot the element tree. Don't write CSS based on Liquid source alone.
2. **One property at a time** — change one CSS rule, push, verify. Don't batch multiple layout changes.
3. **Check specificity** — use browser Computed panel to verify your rules are winning. If not, add `!important`.
4. **Test at multiple widths** — Shopify editor sidebar narrows the preview. Always check at full viewport too.
5. **JS DOM move as last resort** — if CSS can't solve a layout issue due to nesting depth (>2 levels), a 3-line `DOMContentLoaded` script is more reliable than fighting inheritance.

## Liquid Pitfalls

- **`divided_by:` integer truncation.** `{{ 40 | divided_by: 100 }}` outputs `0` (integer division). Use `divided_by: 100.0` (float divisor) to get `0.4`. This silently breaks opacity/percentage calculations.
- **`collection.description` (and other rich text objects) output HTML with `<p>` tags.** If you wrap the output in a `<p>` element — `<p class="desc">{{ collection.description }}</p>` — you get `<p><p>text</p></p>`. HTML spec forbids nested `<p>`, so browsers auto-close the outer `<p>` before the inner one, ejecting the text outside your styled container. The CSS rule on `.desc` then targets an empty element — `getComputedStyle` shows the color IS correct, but `textContent` is empty. **This is extremely deceptive**: DevTools shows the right styles, the element exists, but the visible text is in a sibling `<p>` that has no class. **Fix**: always use `<div>` (never `<p>`) as the wrapper for any Liquid variable that might contain block-level HTML (`collection.description`, `product.description`, `page.content`, `block.settings.body` with richtext type). Add `| strip_html` if you only need plain text. **Diagnostic**: if a styled element appears visually wrong but `getComputedStyle` shows correct values, check `el.textContent` — if it's empty, the content was ejected by HTML auto-correction.

## Section Schema & Setting Type Gotchas

- `settings_schema.json`: empty URL strings (`""`) cause validation failure. **Omit the field** instead of setting it to empty. `theme_support_url` and `theme_support_email` are mutually exclusive — only use one.
- `image_picker` settings reference **Content > Files** (`shopify://shop_images/...`), NOT theme `assets/` folder. These are two completely separate image systems.

### Programmatic Image Upload to Shopify Files CDN

To set `image_picker` values in template JSON programmatically, images must be in Shopify Files (Content > Files), not theme assets. Shopify has no direct upload API — it requires a **three-step staged upload**:

**Prerequisites:** Extract the Shopify CLI session token from `~/Library/Preferences/shopify-cli-kit-nodejs/config.json` → `sessionStore` → `accounts.shopify.com` → `{userId}` → `applications` → `{store}.myshopify.com-{appKey}` → `accessToken`. Use this as `Authorization: Bearer {token}` for all GraphQL calls.

**Step 1 — `stagedUploadsCreate`:** Request a signed upload URL from Shopify.
```bash
curl -s -X POST "https://{store}.myshopify.com/admin/api/2025-01/graphql.json" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"query":"mutation { stagedUploadsCreate(input: [{resource: FILE, filename: \"my-image.jpg\", mimeType: \"image/jpeg\", fileSize: \"89000\", httpMethod: POST}]) { stagedTargets { url resourceUrl parameters { name value } } userErrors { field message } } }"}'
```
Returns a GCS URL + signed parameters. Save `resourceUrl` for Step 3.

**Step 2 — Upload to GCS:** POST the file as multipart form data to the returned `url`, including ALL `parameters` as form fields, with `file=@/path/to/image.jpg` last.
```bash
curl -s -X POST "https://shopify-staged-uploads.storage.googleapis.com/" \
  -F "Content-Type=image/jpeg" \
  -F "success_action_status=201" \
  -F "acl=private" \
  -F "key={value from parameters}" \
  -F "x-goog-date={value}" \
  -F "x-goog-credential={value}" \
  -F "x-goog-algorithm={value}" \
  -F "x-goog-signature={value}" \
  -F "policy={value}" \
  -F "file=@/path/to/my-image.jpg"
```
Expect HTTP 201 with XML response containing `<Location>` and `<ETag>`.

**Step 3 — `fileCreate`:** Register the uploaded file in Shopify Files using the `resourceUrl` from Step 1.
```bash
curl -s -X POST "https://{store}.myshopify.com/admin/api/2025-01/graphql.json" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"query":"mutation { fileCreate(files: [{alt: \"Description\", contentType: IMAGE, originalSource: \"{resourceUrl}\"}]) { files { id fileStatus } userErrors { field message } } }"}'
```
Returns `fileStatus: "UPLOADED"`. The image is now available at `shopify://shop_images/{filename}` for use in `image_picker` settings.

**Common mistakes:**
- Using `resource: IMAGE` instead of `resource: FILE` — `IMAGE` is for product media, `FILE` is for Content > Files.
- Forgetting to include ALL parameters from Step 1 as form fields in Step 2 — missing any one causes 403.
- Trying to use the Shopify CLI `theme push` to upload images to Files — CLI can only push to theme `assets/`, not to Content > Files.
- Using Admin API access tokens from custom apps — the CLI session token (`atkn_2.*`) works via the Partner auth flow, not the standard Admin API key/secret flow.

- **`"type": "video"` vs `"type": "url"` for videos**: Use `"type": "video"` to let merchants pick videos from Shopify CDN in the editor. `"type": "url"` requires pasting a raw link — bad UX. Render with `{{ block.settings.video | video_tag: autoplay: true, muted: true, loop: true }}`.
- **Programmatic video reference format (verified 2026-04)**: To set `"type": "video"` values in template JSON via API, use `"shopify://files/videos/{filename}"`. This is **different** from the image format (`shopify://shop_images/{filename}`). Example: `"desktop_video": "shopify://files/videos/hero-video.mp4"`. Video must first be uploaded via `stagedUploadsCreate` (resource: VIDEO, httpMethod: PUT) + `fileCreate`, and reach `status: READY` before the reference works. Tested 10+ other formats (`shop_videos/`, `videos/`, GID, numeric ID, CDN URL) — all rejected with "must be a valid video shopify url".
- Section schema design must match data source: collection object vs manual product blocks — **decide upfront** before writing any code.
- Section schema with `"tag": null` will be **silently rejected** by Shopify — no error, sections just don't appear. Only valid string values (`"section"`, `"div"`, `"article"`) are accepted; omit the field entirely if not needed. Always run `shopify theme check` before deploying.
- **Preset `"name"` must be a non-empty string.** A preset with `"name": ""` causes Shopify to **silently reject the entire section file** — the file won't be uploaded, templates referencing it will 404, and the push may report "Section type 'xxx' does not refer to an existing section file". This is especially dangerous when batch-cleaning schemas with scripts. **Fix**: always ensure every preset has a meaningful name (e.g. the section's `"name"` value).
- **Select/radio `"default"` must match an option value.** If a `"type": "select"` setting has `"default": ""` but `""` is not in its `options[].value` list, Shopify rejects the section file. **Fix**: always set default to the first option's value, or omit `"default"` entirely (Shopify uses the first option automatically).
- **Header setting `"content"` cannot be empty.** A `"type": "header"` setting with `"content": ""` causes a validation error. **Fix**: always provide a non-empty content string (e.g. `"Settings"`), or remove the header setting.
- **Batch schema cleaning is high-risk.** When using scripts to clear brand-specific defaults from section schemas, these fields are unsafe to blindly set to `""`: preset names, select/radio defaults, header content. **Best practice**: clear only `"type": "text"`, `"type": "textarea"`, `"type": "html"`, `"type": "richtext"` defaults. For select/radio, reset to first option value. For presets, preserve or regenerate the name. Validate ALL sections with `shopify theme push` (or `shopify theme check`) after any batch modification.
- **`"type": "url"` settings in template JSON must use `shopify://` prefix (two slashes), NOT raw paths or three slashes.** When programmatically writing URL values in `index.json`, `page.*.json`, or section group JSON files (`header-group.json`, `footer-group.json`), you must use Shopify's internal resource reference format. Raw paths render as plain text URLs in the theme editor and may bypass Shopify's resource linking system.

  | Format | Result | Valid? |
  |--------|--------|--------|
  | `"/products/my-product"` | Shows raw path in editor, external URL icon | Functional but wrong |
  | `"shopify:///products/my-product"` | Validation error on `theme push` | **Invalid** |
  | `"shopify://products/my-product"` | Shows product name + tag icon in editor | **Correct** |

  Resource URL patterns (all use **two slashes** `shopify://`):
  - Products: `"shopify://products/{handle}"`
  - Collections: `"shopify://collections/{handle}"`
  - Pages: `"shopify://pages/{handle}"`
  - Blogs: `"shopify://blogs/{handle}"`
  - Blog articles: `"shopify://blogs/{blog-handle}/{article-handle}"`

  **How to verify the correct format:** pull the live theme after setting a URL through the theme editor, then inspect the JSON — Shopify saves it in `shopify://` format. Never guess the format; always verify against what Shopify itself produces.

  **Note:** `image_picker` uses a different format: `"shopify://shop_images/filename.jpg"` (with `shop_images` path segment). Don't confuse the two.

## Schema vs Implementation

- **Schema config does NOT equal feature implementation.** Defining a `linklist` or block in schema does nothing if the corresponding HTML/CSS/JS isn't written. Always verify all four layers:
  1. Schema captured (settings appear in theme editor)
  2. HTML renders (Liquid outputs the data)
  3. CSS styles (layout and visuals work)
  4. JS adds interactivity (dropdowns, toggles, etc.)

- **Plan section schema to match the full design upfront.** Every configurable element (text links, image cards, promo blocks) needs its own schema settings/blocks. A 3-column mega menu can't be built from a single-column linklist schema.

## Theme Asset API Deployment

- **Push via API ≠ immediately visible.** Shopify CDN can cache CSS/JS. **Always rename the theme** (version bump, e.g. "Dawn 0.63" → "Dawn 0.64") on every push. This forces CDN cache invalidation and lets the merchant verify the push landed. Without renaming, changes may not appear even after hard refresh.
- **Shopify Speculation Rules causes "invisible" browser prerender cache.** Shopify sends a `speculation-rules` response header that tells Chrome to **prerender** pages via the Speculation Rules API. Once prerendered, the browser uses the cached version without making ANY network request — even `Cmd+Shift+R` (hard refresh) and DevTools `Disable cache` won't clear it. **Symptoms**: Network tab's Doc filter shows ZERO requests after page load; page content is stale; incognito mode shows the updated page correctly. **Diagnosis**: open Network → Doc filter → refresh: if no HTML request appears, the browser is using a prerendered/bfcache page. **Fix**: DevTools → **Application** tab → Storage → **Clear site data** (check all boxes). Or use incognito. **This is the #1 reason "theme push changes don't appear on desktop but work in incognito"** — it's not server cache, not CDN, not HTTP cache; the browser never even sends a request because it has a prerendered version.
- **CDN cache fallback: inline style for critical visual changes.** When debugging color/background/layout changes and version bump still shows stale CSS, add an inline `style` attribute directly in the Liquid template as a temporary fallback: `<div class="your-class" style="background: #xxx;">`. This bypasses CDN entirely. **Remove the inline style after confirming the external CSS is live.** This is especially important for gradient, color, and z-index changes that are hard to visually verify — if you push 3 times and can't tell if it's a CSS bug or CDN cache, the inline style eliminates CDN as a variable.
- **Template JSON files drift between local and remote.** Template files (`index.json`, `product.wuzuo.json`, etc.) get modified by the Shopify theme editor (section content, image selections, block order). Local copies stay as code skeletons. **Before any modification, pull all template JSON from remote via Asset API** and overwrite local. Otherwise you'll destroy editor-configured content.
- **Diff before push, always.** Compare local file checksums against remote API responses. Note that JSON formatting may differ (pretty vs minified) — normalize before comparing.
- **Section group replacement**: Modify `sections/header-group.json` (or `footer-group.json`) to swap Dawn's default header/footer with custom sections. No need to delete the original section files — just change the JSON to reference your custom section type.
- **Debugging flow when something doesn't render**: 1) Verify file content on server via Asset API GET, 2) Check DOM with browser dev tools — element exists?, 3) Check computed styles — anything overridden?, 4) Look for Dawn's `:empty` rules.

## Custom Header/Menu Pitfalls

- **Hamburger → X toggle needs explicit toggle logic.** If you bind hamburger click to `open()` only, clicking the X (which is the same button visually transformed via CSS) just calls `open()` again. **Fix**: use a `toggle()` function that checks `isOpen` state, not separate open/close bindings on the same button.

## Dawn JS Hardcoded Element IDs (Custom Section Breakage)

Dawn's JavaScript (`cart.js`, `cart-drawer.js`, `global.js`, etc.) uses `document.getElementById()` with hardcoded IDs to find and update DOM elements. When you replace a default Dawn section with a custom section, the JS still expects those IDs to exist. If they're missing, the JS throws a `TypeError` → enters `.catch()` → shows a generic error to the user.

**Known hardcoded IDs that Dawn JS depends on:**

| ID | Used by | Purpose |
|----|---------|---------|
| `cart-icon-bubble` | `cart-drawer.js`, `cart.js` | Cart icon in header — updated after every cart change to reflect item count |
| `CartDrawer` | `cart-drawer.js` | Cart drawer container — re-rendered after add-to-cart |
| `CartDrawer-Form` | `cart-drawer.liquid` | The `<form>` inside cart drawer — checkout button uses `form=""` to reference it |
| `CartDrawer-CartErrors` | `cart.js` | Error message container inside cart drawer |
| `CartDrawer-Overlay` | `cart-drawer.js` | Click-to-close overlay |
| `main-cart-items` | `cart.js` | Full cart page items container |
| `main-cart-footer` | `cart.js` | Full cart page footer |

**The pattern:** Dawn's `cart.js` `updateQuantity()` method calls `getSectionsToRender()` which returns a list of `{ id, section, selector }` objects. For each, it does:
```js
document.getElementById(section.id).querySelector(section.selector)
```
If `getElementById` returns `null`, `.querySelector()` throws `TypeError: Cannot read properties of null`. This error is silently caught and surfaced to the user as "There was an error while updating your cart."

**How to prevent:** When building a custom section that replaces a Dawn default (e.g. custom header replacing `header.liquid`), **audit all Dawn JS files for `getElementById` calls that reference elements from the section you're replacing.** Ensure your custom section includes matching IDs. The most common miss is `id="cart-icon-bubble"` on the cart link in custom headers — it must be present for cart drawer quantity updates to work.

**Diagnostic checklist when cart drawer shows "There was an error updating your cart":**
1. Open browser Console → look for `TypeError: Cannot read properties of null`
2. The error trace will point to `cart.js` `updateQuantity()` → `getSectionsToRender().forEach()`
3. Check which `getElementById()` call returns null — that's the missing ID
4. Add the missing ID to your custom section's corresponding element

## `<product-form>` Custom Element — 5 Hidden Dependencies (Verified 2026-04)

When replacing Dawn's `main-product.liquid` with a custom product section, the `<product-form>` custom element silently breaks unless ALL 5 implicit dependencies are satisfied. Each failure mode is silent — no console error, no visible error, the button just "doesn't work."

**Dependency checklist (all required):**

| # | Dependency | What breaks without it | Fix |
|---|-----------|----------------------|-----|
| 1 | `product-form.js` loaded | `<product-form>` is plain HTML, no AJAX | `<script src="{{ 'product-form.js' | asset_url }}" defer></script>` |
| 2 | `Shopify.routes.cart_add_url` defined | `fetch(undefined)` → silent failure | Inline `<script>` setting `window.Shopify.routes.cart_add_url = '{{ routes.cart_add_url }}'` |
| 3 | Script loaded with `defer` | Constructor runs before children parsed → `querySelector('form')` = null → constructor crashes silently | Use `defer` attribute, NOT `script_tag` filter (which is synchronous) |
| 4 | Submit button text wrapped in `<span>` | `submitButtonText = btn.querySelector('span')` = null → crash when setting loading text | `<button><span>Add to Cart</span></button>` |
| 5 | `.loading__spinner` element inside button | `querySelector('.loading__spinner').classList.remove('hidden')` → null.classList → TypeError → AJAX aborted | Add `<div class="loading__spinner hidden"><svg>...</svg></div>` inside button |

**Why these are hard to debug:**
- Custom element constructors swallow errors silently (spec behavior)
- `onSubmitHandler` errors are caught by Dawn's try/catch → no console output
- Each dependency fails at a DIFFERENT stage (registration → construction → submit → AJAX), so fixing one reveals the next
- The button appears to "do nothing" — no error, no redirect, no feedback

**Minimal working `<product-form>` template:**
```liquid
<script>
  window.Shopify = window.Shopify || {};
  window.Shopify.routes = window.Shopify.routes || {};
  window.Shopify.routes.cart_add_url = '{{ routes.cart_add_url }}';
  window.Shopify.routes.cart_change_url = '{{ routes.cart_change_url }}';
  window.Shopify.routes.cart_update_url = '{{ routes.cart_update_url }}';
  window.Shopify.routes.cart_url = '{{ routes.cart_url }}';
</script>

<!-- product-form element with all required children -->
<product-form>
  {%- form 'product', product, data-type: 'add-to-cart-form', novalidate: 'novalidate' -%}
    <input type="hidden" name="id" value="{{ variant.id }}">
    <button type="submit" name="add">
      <span>Add to Cart</span>
      <div class="loading__spinner hidden">
        <svg viewBox="0 0 66 66" width="18" height="18">
          <circle fill="none" stroke-width="6" cx="33" cy="33" r="30" stroke="currentcolor" opacity=".25"/>
          <circle fill="none" stroke-width="6" cx="33" cy="33" r="30" stroke="currentcolor" stroke-dasharray="164" stroke-dashoffset="130"/>
        </svg>
      </div>
    </button>
  {%- endform -%}
</product-form>

<!-- MUST be after product-form element in DOM -->
<script src="{{ 'product-form.js' | asset_url }}" defer="defer"></script>
```

**Diagnostic script (run in Console when Add to Cart doesn't work):**
```js
(function(){
  var pf = document.querySelector('product-form');
  if (!pf) return console.log('FAIL: no <product-form> element');
  console.log('registered:', !!customElements.get('product-form'));
  console.log('pf.form:', pf.form || 'UNDEFINED — constructor failed');
  console.log('pf.cart:', pf.cart?.tagName || 'UNDEFINED — no cart-drawer/notification');
  console.log('onSubmitHandler:', typeof pf.onSubmitHandler);
  console.log('btn > span:', pf.querySelector('[type=submit] span') ? 'OK' : 'MISSING');
  console.log('loading__spinner:', pf.querySelector('.loading__spinner') ? 'OK' : 'MISSING');
  console.log('routes.cart_add_url:', window.Shopify?.routes?.cart_add_url || 'UNDEFINED');
})();
```
All 7 lines should show truthy values. The first `UNDEFINED`/`MISSING` is your bug.

## Static-to-Theme Migration

Recommended order for e-commerce theme projects:

1. Build static HTML/CSS/JS prototype first (fast iteration, no Shopify preview delay)
2. Split into Shopify sections/snippets/templates
3. Replace hardcoded data with Liquid variables
4. Create real product data via API
5. Deploy theme and verify

Static prototyping is much faster than iterating directly in Liquid/Shopify preview. Get the design right first, then convert.

## CSS Unit Conversion: Identify the Source Unit System BEFORE Converting

- **Converting CSS values without understanding the source unit system produces values that only match at one viewport width.** This is the #1 cause of "looks right on my screen but wrong on the client's screen" in static-to-theme migrations.
- **Step 0 of any migration: run `getComputedStyle(document.documentElement).fontSize` on the source site at multiple viewport widths.** If the value CHANGES with viewport, the source uses a fluid unit system. If it's fixed (e.g. always `16px` or always `10px`), it uses a fixed unit system.

**Three common source unit systems and how to convert:**

| Source system | How to detect | Convert to |
|--------------|---------------|------------|
| **Fixed rem** (standard `1rem=16px`) | `html fontSize` = `16px` at all viewports | `px` (multiply rem × 16) |
| **Fixed rem** (Dawn-style `1rem=10px`) | `html fontSize` = `10px` at all viewports | `px` (multiply rem × 10) |
| **Fluid rem** (viewport-scaled) | `html fontSize` changes with viewport (e.g. `7.5px` at 1200px, `10px` at 1600px) | `vw` (see formula below) |

**Fluid rem → vw conversion formula:**
If the source uses `font-size: calc((100vw / BASE) * 10)` where `BASE` is the design width (commonly 1600px or 1440px):
```
N rem = N × 10 / BASE × 100 vw = N × 1000 / BASE vw

Examples (BASE = 1600):
  6rem → 6 × 1000 / 1600 = 3.75vw
  4rem → 4 × 1000 / 1600 = 2.5vw
  1.5rem → 1.5 × 1000 / 1600 = 0.9375vw
```

**Why `px` is wrong for fluid sources:** `6rem` in a fluid-1600 system = `45px` at 1200px viewport, `60px` at 1600px, `75px` at 2000px. Using fixed `60px` matches ONLY at 1600px — too wide on small screens, too narrow on large screens.

**Diagnostic script (run on source site):**
```js
// Run at different viewport widths (resize browser) to detect fluid vs fixed
console.log('fontSize:', getComputedStyle(document.documentElement).fontSize,
            'viewport:', window.innerWidth,
            'ratio:', parseFloat(getComputedStyle(document.documentElement).fontSize) / window.innerWidth);
```
If `ratio` is constant across viewports → fluid system. If `fontSize` is constant → fixed system.

**This applies to ALL migrated properties** — not just padding. Any CSS value originally authored in the source's rem (margins, gaps, max-widths, font-sizes, positions) must be converted using the correct target unit. Mixing `vw` for spacing and `px` for font-sizes is fine — font-sizes often have `clamp()` or explicit responsive breakpoints in the source that make `px` appropriate.

## Collection Filter `value.url` Is Empty — Use `param_name` + `value` Instead

- **`filter_value.url` and `filter_value.url_to_remove` can render as empty strings.** When building custom collection filter UI with `collection.filters`, `{{ value.url }}` often outputs nothing. Dawn's `facets.liquid` **never uses `value.url`** — it uses `value.param_name` + `value.value` inside a `<form>` or constructs URLs in JS. **Fix**: use `data-param="{{ value.param_name }}"` and `data-value="{{ value.value }}"` on each filter element, then build the URL in JavaScript with `url.searchParams.append(param, val)`. Also store `data-active="{{ value.active }}"` so JS knows whether to add or remove the filter.
- **This is the #1 cause of "filters render but don't work"** in custom collection sections. Always check rendered HTML for `data-url=""` — if empty, switch to `param_name` + `value`.

## Price Range Filter: URL Params Use Dollars, Liquid Returns Cents

- **Shopify storefront `filter.v.price.gte` / `filter.v.price.lte` URL params expect DOLLAR values, NOT cents.** `filter.v.price.lte=100` means "max $100". But Shopify's Liquid `filter.range_min`, `filter.range_max`, `filter.min_value.value`, and `filter.max_value.value` all return values in the **smallest currency subunit** (cents for USD). This unit mismatch is the #1 cause of "price filter renders but doesn't actually filter products".
- **Symptoms**: slider UI works visually, URL changes on drag, but ALL products still show regardless of price range. The price label may show absurd values like "$10000.00" when the max product is $268.
- **The trap**: if your slider values are in dollars (0–268 for $0–$268), and your JS multiplies by 100 before setting URL params (`value * 100`), the URL sends `filter.v.price.lte=26800`. Shopify interprets this as **$26,800** (not $268), so every product passes the filter.
- **Correct pattern**:
  ```
  Liquid (cents → dollars for slider):
    range_max_dollars = filter.range_max | divided_by: 100       ← slider max="268"
    cur_max_dollars   = filter.max_value.value | divided_by: 100 ← slider value="100" when active

  JS (dollars → dollars for URL, NO multiplication):
    var vMax = parseInt(maxInput.value);  // already in dollars
    url.searchParams.set('filter.v.price.lte', vMax);
  ```
- **Diagnostic**: apply `?filter.v.price.lte=100` manually to the collection URL. If products over $100 disappear, the URL unit is dollars. If ALL products still show, the value is being treated as cents (meaning a different problem — check Search & Discovery app configuration).
- **Also check**: the Shopify **Search & Discovery** app must be installed and the price filter must be enabled in **Online Store → Navigation → Collection and search filtering** for any price filtering to work. If `collection.filters` renders the price slider but filtering has no effect even with correct URL params, the app/configuration is the issue, not the theme code.

## Replacing Dawn's Price Inputs with a Custom Range Slider (verified 2026-04)

- **Dawn's `<price-range>` custom element aggressively hijacks all `<input>` events inside it.** Its `onRangeChange` → `adjustToValidValues()` intercepts `input` events, clamps values, and triggers form submission. Any custom slider placed inside `<price-range>` will have its values reset to 0 on drag. **This is the #1 cause of "slider jumps to 0 when dragged".**

- **Fix: Kill Dawn's methods at the PROTOTYPE level, not the instance level.**
  ```js
  // ❌ WRONG — instance override gets lost when element re-upgrades
  var pr = document.querySelector('price-range');
  pr.onRangeChange = function(){};

  // ✅ RIGHT — prototype override survives all instances
  customElements.whenDefined('price-range').then(function(){
    var PR = customElements.get('price-range');
    PR.prototype.onRangeChange = function(){};
    PR.prototype.adjustToValidValues = function(){};
    PR.prototype.setMinAndMaxValues = function(){};
  });
  ```
  Instance-level overrides fail because: (1) `customElements.whenDefined` fires when the CLASS is registered, not when each INSTANCE is upgraded; (2) the constructor runs after your override and re-sets the methods.

- **Script timing: must wait for custom element registration.**
  ```js
  // Dawn's global.js loads with defer — custom elements register AFTER inline scripts run
  if(customElements.get('price-range')){
    killAndInit();
  } else {
    customElements.whenDefined('price-range').then(killAndInit);
  }
  // Fallback
  document.addEventListener('DOMContentLoaded', function(){ setTimeout(init, 200); });
  ```

- **Use URL navigation, not Dawn's form submission.** Dawn's AJAX section rendering is tightly coupled to its own input elements. Instead of syncing values to hidden Dawn inputs, just navigate directly:
  ```js
  function go(){
    var url = new URL(window.location.href);
    url.searchParams.delete('filter.v.price.gte');
    url.searchParams.delete('filter.v.price.lte');
    if(min > 0) url.searchParams.set('filter.v.price.gte', min);
    if(max < rangeMax) url.searchParams.set('filter.v.price.lte', max);
    window.location.href = url.toString();
  }
  ```
  Full page reload is acceptable — price filtering is a low-frequency action.

- **Dual range slider z-index trick for overlapping thumbs.** Two `<input type="range">` stacked via `position:absolute` need `pointer-events:none` on the input + `pointer-events:auto` on the thumb. The max slider should have higher z-index by default; when the min thumb passes 50%, swap z-index so it stays grabbable:
  ```css
  .range--min { z-index:3 }
  .range--max { z-index:4 }
  .range--min.is-upper { z-index:5 }  /* JS toggles this class */
  ```

- **Liquid: `nil | divided_by: 100` returns 0, and `default` does NOT override 0.** When no price filter is active, `filter.max_value.value` is nil → `nil | divided_by: 100` = 0. The `default` filter only triggers on nil/empty/false, NOT on 0. Use `unless` instead:
  ```liquid
  {%- assign cur_max = filter.max_value.value | divided_by: 100 -%}
  {%- unless cur_max > 0 -%}{%- assign cur_max = range_max -%}{%- endunless -%}
  ```

- **Complete snippet architecture** (`snippets/price-facet.liquid`):
  1. Liquid: compute `range_max`, `cur_min`, `cur_max` (all in dollars, `divided_by: 100`)
  2. HTML: two `<input type="range">` + display spans + fill bar (NO Dawn inputs needed)
  3. CSS: track + fill + thumb styling, hide Dawn's `.field` and `.field-currency`
  4. JS: `customElements.whenDefined` → kill prototype → init sliders → `draw()` on input, `go()` on change

## Mobile Filter Drawer Must Be Explicitly Built

- **Desktop sidebar filters do NOT automatically work on mobile.** You must build the mobile drawer yourself: slide-up panel, overlay, close button, body scroll lock, and JS event bindings.
- **Use event delegation for filter clicks inside drawers.** Binding `change` on hidden checkboxes (`opacity: 0; width: 0`) is unreliable on mobile. Use `document.addEventListener('click', ...)` with `.closest('.filter-item')` instead.
- **Mobile drawer z-index must be lower than the mobile menu.** If the filter toolbar uses `position: sticky`, give its `.shopify-section` wrapper `position: relative; z-index: 1` to prevent it from floating above the mobile nav menu.

## Shopify Admin API — Store Operations

These are Admin API operations commonly needed during theme development (product data, publishing, markets, shipping). Requires an access token — see `ed-shopify-api-rules` for full OAuth flow.

### Publishing Products to Storefront

- Products created via API are **unpublished by default** — they won't render in Liquid (`collection.products` returns empty) or appear in the storefront until explicitly published.
- Use `publishablePublish` mutation with a **Publication ID** (`gid://shopify/Publication/...`), NOT a Channel ID (`gid://shopify/Channel/...`). These are different objects.
- **Get Publication ID first:**
  ```graphql
  { publications(first: 10) { edges { node { id name } } } }
  ```
  Look for the one named "Online Store".
- **Then publish:**
  ```graphql
  mutation { publishablePublish(
    id: "gid://shopify/Product/xxx",
    input: [{ publicationId: "gid://shopify/Publication/xxx" }]
  ) { publishable { ... on Product { title } } userErrors { field message } } }
  ```

### 多国市场定价（Markets & Price Lists）

通过 Price Lists 为不同市场设置固定价格：

```
1. 查询市场对应的 Price List：
   catalogs(first: 20, type: MARKET) → MarketCatalog → priceList { id currency }

2. 为变体设置固定价格：
   mutation { priceListFixedPricesAdd(
     priceListId: "gid://shopify/PriceList/xxx",
     prices: [{
       variantId: "gid://shopify/ProductVariant/xxx",
       price: { amount: "100", currencyCode: JPY }
     }]
   ) { prices { ... } userErrors { ... } } }

3. 可并行为多个市场设置价格（每个市场一个 mutation call）
```

**注意**：需要 `write_markets` scope，否则报 ACCESS_DENIED。

### Shipping Zones（物流区域）

通过 `deliveryProfileUpdate` mutation 管理，**不要用 REST** `shipping_zones.json`（POST 返回空响应，DELETE 看似成功但不生效）。

```
1. 查询当前 delivery profile：
   deliveryProfiles(first: 5) → profileLocationGroups → locationGroupZones → zone { id name countries }

2. 删除旧 zone + 创建新 zone（一次 mutation）：
   mutation deliveryProfileUpdate(id: $profileId, profile: {
     zonesToDelete: ["gid://shopify/DeliveryZone/xxx"],
     locationGroupsToUpdate: [{
       id: "gid://shopify/DeliveryLocationGroup/xxx",
       zonesToCreate: [{
         name: "Continental US (Free Shipping)",
         countries: [{ code: "US", provinces: [{ code: "AL" }, { code: "CA" }, ...] }],
         methodDefinitionsToCreate: [{
           name: "Free Standard Shipping",
           active: true,
           rateDefinition: { price: { amount: 0, currencyCode: "USD" } }
         }]
       }]
     }]
   })
```

**关键细节**：
- `zonesToDelete` 在 `profile` 顶层，不在 `locationGroupsToUpdate` 内部
- `countries.provinces` 可精确控制州/省，用于排除 AK/HI/领地等

### Markets（市场创建/管理）

```
1. 查询现有市场：
   markets(first: 10) → id, name, enabled, primary, regions → code, name

2. 创建市场：
   mutation marketCreate(input: {
     name: "United States",
     enabled: true,
     regions: [{ countryCode: US }]   ← 注意是 countryCode 不是 code
   })

3. 删除市场：
   mutation marketDelete(id: "gid://shopify/Market/xxx")
```

**Primary Market 限制（已验证 2026-03）**：
- `MarketUpdateInput` 没有 `primary` 字段，**无法通过 API 切换主市场**
- `marketRegionsCreate` 对主市场报错："Cannot add regions to the primary market"
- 主市场跟店铺注册地址绑定，只能在后台 **Settings > Store details** 改地址或 **Settings > Markets** 手动切换
- 非主市场可以正常通过 API 创建/删除/启用/禁用

## CLI Theme Token vs OAuth App Token

Shopify CLI 的 theme session token（`atkn_2.*`）和 OAuth app token（`shpca_*`）权限完全不同：

| 能力 | CLI theme token | OAuth app token |
|------|----------------|-----------------|
| Theme push/pull | ✅ | ✅ |
| 产品 CRUD | ✅ | ✅ |
| 图片上传（stagedUploads + fileCreate） | ✅ | ✅ |
| **博客文章 CRUD** | ❌ | ✅ |
| **导航菜单 CRUD** | ❌ | ✅ |
| **政策设置** | ❌ | ✅ |
| **库存管理** | ❌ | ✅ |
| 位置查询（locations） | ❌ | ✅ |

**CLI theme token** 存储在 `~/Library/Preferences/shopify-cli-kit-nodejs/config.json`，有效期短（几小时），只有 `shop.admin.themes` 相关 scope。

**OAuth app token** 通过自建 Custom App 的 OAuth 流程获取，scope 由 app 配置决定，持久有效。**所有非 theme 的 Admin API 操作（菜单、博客、政策、库存）都必须用 OAuth app token。**

### 导航菜单管理（menuUpdate）

通过 GraphQL `menuUpdate` mutation 管理菜单。**没有 `menuItemDelete`** — 只能用 `menuUpdate` 重写整个菜单的 items 列表。

```graphql
# 查询所有菜单
{ menus(first:10) { edges { node { id title handle items { id title url } } } } }

# 更新菜单（重写全部 items）
mutation { menuUpdate(
  id: "gid://shopify/Menu/xxx",
  title: "Footer Help",
  items: [
    { title: "Shipping", type: SHOP_POLICY, resourceId: "gid://shopify/ShopPolicy/xxx" },
    { title: "Blog", type: BLOG, resourceId: "gid://shopify/Blog/xxx" },
    { title: "Our Story", type: HTTP, url: "https://store.myshopify.com/pages/our-story" }
  ]
) { menu { id items { title } } userErrors { field message } } }
```

**`items` 中每项必须有 `type` 字段**（`MenuItemType` 枚举）：
- `HTTP` — 任意 URL（需要 `url` 字段）
- `SHOP_POLICY` — 政策页（需要 `resourceId`，如 `gid://shopify/ShopPolicy/xxx`）
- `COLLECTION` — 集合（需要 `resourceId`）
- `PRODUCT` — 产品
- `PAGE` — 页面
- `BLOG` — 博客
- `ARTICLE` — 文章
- `FRONTPAGE` — 首页
- `SEARCH` — 搜索页

### 博客文章创建

使用 REST Admin API（GraphQL 的 `articleCreate` 较新，REST 更稳定）：

```bash
# 先查 blog ID
curl -s "https://{store}.myshopify.com/admin/api/2025-01/blogs.json" \
  -H "X-Shopify-Access-Token: {shpca_token}"

# 创建文章
curl -s -X POST "https://{store}.myshopify.com/admin/api/2025-01/blogs/{blog_id}/articles.json" \
  -H "Content-Type: application/json" \
  -H "X-Shopify-Access-Token: {shpca_token}" \
  -d '{
    "article": {
      "title": "Article Title",
      "body_html": "<p>Article content...</p>",
      "tags": "Buying Guide",
      "published": true
    }
  }'

# 设置封面图（base64 内联）
curl -s -X PUT "https://{store}.myshopify.com/admin/api/2025-01/articles/{article_id}.json" \
  -H "Content-Type: application/json" \
  -H "X-Shopify-Access-Token: {shpca_token}" \
  -d '{"article":{"id":{article_id},"image":{"attachment":"{base64_encoded_image}"}}}'
```

### 店铺政策设置

使用 GraphQL `shopPolicyUpdate` mutation：

```graphql
# 查询现有政策
{ shop { shopPolicies { id type title } } }

# 创建/更新政策
mutation { shopPolicyUpdate(shopPolicy: {
  type: REFUND_POLICY,    # PRIVACY_POLICY, SHIPPING_POLICY, TERMS_OF_SERVICE, SUBSCRIPTION_POLICY
  body: "<h2>Refund Policy</h2><p>...</p>"
}) { shopPolicy { id } userErrors { field message } } }
```

**注意**：REST API `GET /policies.json` 是只读的，无法通过 REST 创建/更新政策。必须用 GraphQL。

## Development Store API 推送限流（已验证 2026-04）

- **短时间内通过 Asset API 密集推送文件（10+ 次/分钟）会触发 Shopify 渲染限流。** Development 计划的 store 在密码保护模式下尤其敏感。**症状**：推完后刷新页面间歇性 503（"There was a problem loading this website"），Theme Editor 报 "Failed to load the page"，但等几分钟后自动恢复。**这不是代码错误** — 是 Shopify 的 Liquid 渲染队列在消化密集推送。
- **容易误诊为 Liquid 崩溃**：503 错误页面和真正的 Liquid 语法错误（500）外观完全一样，都显示 "There was a problem loading this website"。区别方式：Liquid 语法错误是 **100% 复现**的（每次刷新都崩），限流是 **间歇性**的（刷几次能成功）。
- **防止触发限流的最佳实践**：
  1. 批量改多个文件时，**每次推送间隔 3-5 秒**（`sleep 3` between API calls）
  2. 一次会话中 Asset API PUT 次数尽量控制在 **15 次以内**
  3. 合并改动 — 能一次推的不要分两次（如 CSS 改动合并后一次推）
  4. 推完后 **不要立刻反复刷新 Editor** — 每次刷新都触发一次完整 Liquid 渲染
  5. Version bump 放在最后一次推送之后，不要每次改文件都 bump
- **触发后的恢复**：通常 2-5 分钟自动恢复。无需任何操作。如果超过 10 分钟仍然 503，才需要检查代码问题。

## Product Page (PDP) Standard Block Architecture (verified 2026-04)

Every e-commerce Shopify theme project should have these blocks in the product section. This is the reusable blueprint — copy and adapt per project.

### Block inventory (12 types, 3 categories)

**Category A — Auto blocks (limit:1, no merchant input needed):**
Read directly from `product` object. Merchant can reorder/hide but doesn't fill content.

| Block type | Schema name | What it renders |
|---|---|---|
| `title` | Title | `product.title` + badges (from `product.type`, `product.tags`, `metafields.custom.badge`) + subtitle (`metafields.custom.subtitle`) |
| `price` | Price | `current_variant.price \| money` |
| `description` | Description | `product.description` |
| `variant_picker` | Variant Picker | Variant buttons (auto-hides for single-variant products). Updates hidden `<input name="id">` via JS |
| `quantity_selector` | Quantity Selector | +/- buttons + readonly input. Syncs to form quantity |
| `buy_buttons` | Buy Buttons | Primary CTA (Shop Now → `/cart/add` POST → checkout) + Secondary CTA (Add to Cart → `<product-form>` AJAX → cart drawer) |

**Category B — Configurable blocks (limit:1, merchant customizes):**

| Block type | Schema name | Settings |
|---|---|---|
| `trust_badges` | Trust Badges | 4 slots, each: `icon_image` (image_picker) + `icon_svg` (html) + `label` (text). Renders as 4-column grid |
| `icon_with_text` | Icon with Text | `icon_image` (image_picker) + `icon_svg` (html) + `text` (text). Single line with icon. No limit — can add multiple |
| `text_block` | Text Block | `content` (richtext). For discount tiers, notes, etc. No limit — can add multiple |
| `secure_checkout` | Secure Checkout | `heading` (text) + 6 payment icon slots (each: `name` + `icon` image_picker + `svg` html) + `note` (html for SSL/PCI text) |

**Category C — Repeatable blocks (no limit):**

| Block type | Schema name | Settings |
|---|---|---|
| `inline_video` | Product Video | `video` (video) + `poster` (image_picker). Renders in gallery area |
| `accordion_item` | Accordion Item | `title` (text) + `content` (richtext). For specs, craft process, shipping info, etc. |

### Implementation pattern

```liquid
{%- for block in section.blocks -%}
  {%- case block.type -%}
    {%- when 'title' -%}
      <div {{ block.shopify_attributes }}>
        <h1>{{ product.title }}</h1>
      </div>
    {%- when 'price' -%}
      <div {{ block.shopify_attributes }}>{{ current_variant.price | money }}</div>
    {%- when 'trust_badges' -%}
      {%- for i in (1..4) -%}
        {%- assign lbl = block.settings['label_' | append: i] -%}
        {%- if lbl != blank -%}...render badge...{%- endif -%}
      {%- endfor -%}
    ...
  {%- endcase -%}
{%- endfor -%}
```

Key rules:
- **Every block must include `{{ block.shopify_attributes }}`** — this enables Editor click-to-select
- **Auto blocks use `limit: 1`** — prevents merchants from accidentally duplicating title/price
- **Icon settings always offer dual input**: `image_picker` (upload) + `html` (SVG paste), with priority: image > SVG > default fallback
- **Trust badges / Secure checkout use numbered settings** (`label_1`, `icon_image_1`, etc.) inside a single block, NOT separate blocks per item — keeps the Editor sidebar clean
- **`buy_buttons` block must contain both Shop Now (plain form POST) and Add to Cart (`<product-form>` AJAX)** — see `<product-form>` 5 hidden dependencies section in this document

### Product recommendations section

Use `<product-recommendations>` custom element to fetch Shopify's AI recommendations:

```liquid
<product-recommendations data-url="{{ routes.product_recommendations_url }}?section_id={{ section.id }}&product_id={{ product.id }}&limit=8&intent=related">
  {%- if recommendations.performed and recommendations.products_count > 0 -%}
    {%- assign products = recommendations.products -%}
  {%- else -%}
    {%- assign products = product.collections.first.products -%}
  {%- endif -%}
  ... render cards ...
</product-recommendations>
```

The custom element JS fetches the URL on `connectedCallback`, replaces its own innerHTML with the response. Fallback renders same-collection products for SSR/initial load. Must re-bind any JS (carousel arrows, etc.) after AJAX replacement.
