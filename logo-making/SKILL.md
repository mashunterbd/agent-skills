---
name: logo-system
description: Turn one supplied logo (SVG, PNG, JPG, WebP, PDF, screenshot) into a complete web brand-asset system and install it in any website or web app. Produces theme-adaptive SVG logos (light/dark/auto via currentColor, CSS variables), favicon.ico, icon.svg with dark mode, Apple touch icon, PWA/Android icons incl. maskable, web manifest, Open Graph/Twitter share images, email PNGs, Schema.org logo markup, and framework-specific integration code for any stack (HTML, React, Next.js, Vue, Nuxt, Svelte, Astro, Angular, Laravel, Django, Rails, WordPress, Shopify, etc.). Use whenever the user provides a logo and mentions favicon, app icon, PWA icon, brand assets, dark mode logo, OG/social share image, apple-touch-icon, or 'add my logo to the website', even if they do not list every asset. Also use to audit or fix an existing site's logo/icon setup.
---

# Logo System

Take one logo from a human and produce every asset a modern website needs, then install those assets correctly in whatever stack the project uses. The output should match what large tech companies ship: vector-first, theme-aware, crisp at 16px, and complete across browsers, phones, search engines, and social platforms.

## Why this skill exists

A single logo file is not enough for the web. Browsers want a tiny simplified icon, iOS wants an opaque 180px square, Android wants a maskable icon with a safe zone, social platforms want a 1200x630 card, and dark mode needs the logo to stay visible. Raster files (PNG/JPG) cannot be recolored with CSS, so the whole system is built on SVG masters. Doing this by hand takes hours and is easy to get subtly wrong (blurry favicons, cropped Android icons, invisible dark-on-dark logos). This skill makes the process repeatable.

## Core principles

1. **Fidelity first.** The final logo must look like the user's logo. Never redesign, "improve", recolor the brand, change proportions, or swap the typeface unless the user asks. Derived variants (mono, small-size) simplify the logo but stay recognizably the same.
2. **Vector master, raster derivatives.** Every PNG/ICO is rendered from an SVG. Never upscale a small raster.
3. **Theme-awareness through code, not duplicate files**, wherever possible: `currentColor`, CSS custom properties, `prefers-color-scheme`. Separate light/dark files are the fallback for logos whose dark and light designs genuinely differ.
4. **Detect before you write.** Inspect the project (framework, router, public directory, existing icons, CSP, base path) and integrate in that stack's idiom. Do not drop a Next.js `<head>` snippet into a Vue project.
5. **Verify visually.** Render the outputs, look at them, and compare them with the source logo. Report anything you could not guarantee.

## Workflow

Follow these phases in order. Skip a phase only when it clearly does not apply.

### Phase 1 - Intake and analysis

Locate the logo and classify it.

| Input | What to do |
|---|---|
| SVG | Best case. Inspect and clean (Phase 2). |
| PDF / AI / EPS | Convert to SVG (`pdftocairo -svg`, `inkscape --export-type=svg`, or `rsvg-convert`). Outline all text. |
| PNG / JPG / WebP | Needs vectorization (Phase 2B). Ask the user for the original vector file once, briefly; proceed if they do not have one. |
| Screenshot / photo of a logo | Same as raster, but crop tightly and warn the user that fidelity depends on image quality. |
| Only a description or name | Ask for the file. Do not invent a logo. |

Then record these facts (write them in a short `brand-analysis` note for yourself, show the user a summary):

- **Logo type:** symbol only, wordmark only, combination (symbol + wordmark), emblem/badge.
- **Aspect ratio** of the full logo and of the symbol alone.
- **Colors:** exact hex values of every fill/stroke, plus gradients. Identify the primary brand color.
- **Has transparency?** What background was it designed for (light, dark, colored)?
- **Complexity at 16px:** does the symbol survive being tiny? Thin strokes, gradients, small text, and fine detail usually do not.
- **Contrast on light (#fff) and dark (#0b0b0b) backgrounds:** compute WCAG contrast of each brand color. Below 3:1 on either background means that variant needs a color adjustment or an outline (see Appendix B).

### Phase 2 - Build the SVG master set

#### 2A. Vector source available

1. Open the SVG and normalize it: set a clean `viewBox`, remove fixed `width`/`height`, delete editor metadata, convert any `<text>` to paths (fonts will not exist on visitors' machines), flatten transforms where practical, convert hard-coded strokes to fills only if appearance is identical.
2. Run SVGO if available (`npx svgo --multipass`), but keep `viewBox` and do not let it strip IDs that gradients/masks reference. Compare renders before and after.
3. Crop the `viewBox` tightly to the artwork, then add padding only where a specific asset needs it (the generator script handles this).

#### 2B. Raster only: vectorize faithfully

1. Pick the cleanest, largest version available. Remove any background (flood-fill white/solid backgrounds to transparent).
2. Trace with a real tracer: `vtracer` (`pip install vtracer`) for multi-color logos, `potrace` for single-color. Tune for clean curves (low speckle filter, reasonable corner threshold, color precision matched to the number of real brand colors).
3. For simple geometric logos (circles, rounded squares, basic letterforms), hand-writing clean SVG shapes usually beats tracing. Do this when you can match the original exactly.
4. **Quantify fidelity:** render the SVG at the raster's size and compute a pixel difference or SSIM against the original. Eyeball a side-by-side. If it is visibly off (wobbly curves, lost counters in letters, wrong colors), iterate or tell the user plainly that a proper vector file from their designer is recommended.
5. Never claim a trace is "identical". Say "traced, matches closely" and give the measured difference.

#### Variants to produce (SVG masters)

Save into `brand/svg/` (or the project's convention). See Appendix A for exact specs.

| File | Purpose |
|---|---|
| `logo.svg` | Full-color primary logo, original colors |
| `logo-mono.svg` | Single shape color using `fill="currentColor"`, inherits text color; best default for headers and footers |
| `logo-light.svg` | For use on light backgrounds (dark-ish artwork) |
| `logo-dark.svg` | For use on dark backgrounds (light-ish artwork) |
| `logo-horizontal.svg` | Symbol + wordmark side by side (if the logo has both) |
| `logo-stacked.svg` | Symbol above wordmark (if the logo has both) |
| `icon.svg` | Symbol only, square viewBox, simplified for small sizes, with built-in `prefers-color-scheme` dark mode. Used as the browser favicon |
| `icon-mono.svg` | Symbol only, `currentColor` |
| `safari-pinned-tab.svg` | Single-color flat symbol (legacy Safari; still harmless to ship) |

If the logo is symbol-only, skip horizontal/stacked. If it is a wordmark only, `icon.svg` should be a monogram (first letter or the existing short mark), drawn from the original letterform, not a new design. Tell the user which letterform you used and why.

**Small-size simplification rule:** the favicon version may drop fine detail, thicken strokes, remove tiny text/taglines, and enlarge the symbol to fill the square, but must keep the same shape language and brand color. Test at 16x16 and 32x32 before accepting.

### Phase 3 - Generate raster and config assets

Render everything from the SVG masters in one deterministic pass using the generator script in Appendix D. Save it as `scripts/generate_assets.py` in the project (or a temp folder) first, then run:

```bash
pip install cairosvg pillow --break-system-packages   # if missing
python scripts/generate_assets.py \
  --icon brand/svg/icon.svg \
  --logo brand/svg/logo.svg \
  --out public \
  --name "Acme" \
  --short-name "Acme" \
  --theme-color "#0F62FE" \
  --bg-light "#FFFFFF" \
  --bg-dark "#0B0B0B"
```

Run it with `--help` for all options. The script produces:

- `favicon.ico` (16, 32, 48 embedded), `favicon-16x16.png`, `favicon-32x32.png`
- `apple-touch-icon.png` (180x180, opaque background, padded)
- `icon-192.png`, `icon-512.png`, `icon-maskable-512.png` (inside the 80% safe zone), plus `mstile-150x150.png`
- `logo@1x.png`, `logo@2x.png`, `logo@3x.png` (transparent, header-sized) and `logo-2048.png` (high-res backup)
- `og-image.png` (1200x630) and `twitter-image.png` (1200x675), logo centered on a brand-colored or neutral canvas
- `site.webmanifest` with correct icon entries and `theme_color` / `background_color`
- `head-snippet.html`, a ready-to-paste block of `<link>` and `<meta>` tags

If the script cannot run (missing system libs), fall back to `sharp` (Node), `rsvg-convert` + ImageMagick, or Inkscape CLI. The required outputs are the contract; the tool is flexible. For ICO without the script: `magick icon-16.png icon-32.png icon-48.png favicon.ico`.

Note: if `icon.svg` already contains its own filled background shape (for example a rounded square), the maskable icon will show that shape scaled down on the `--icon-bg`/theme color. For the cleanest Android result, pass a symbol-only SVG (no background plate) via a separate run, or set `--icon-bg` to the plate's color.

Review rendered outputs by viewing them. Specifically check 16x16 and 32x32 enlarged, the maskable icon (nothing important near edges), the apple-touch icon (no transparent corners), and the OG image (logo legible, not tiny).

### Phase 4 - Detect the stack and integrate

Detect the framework from project files before touching code. Use Appendix C for the exact integration recipe for each stack. Quick detection guide:

| Signal | Stack |
|---|---|
| `next.config.*`, `app/` or `pages/` | Next.js (App Router vs Pages Router differ) |
| `nuxt.config.*` | Nuxt |
| `svelte.config.*` | SvelteKit |
| `astro.config.*` | Astro |
| `angular.json` | Angular |
| `vite.config.*` + `src/main.(tsx|jsx|ts|js)` | Vite + React/Vue/Svelte/Solid |
| `remix.config.*` / `app/root.tsx` | Remix / React Router |
| `artisan`, `resources/views` | Laravel (Blade) |
| `manage.py`, `templates/` | Django |
| `Gemfile`, `config/routes.rb` | Rails |
| `wp-content`, `functions.php` | WordPress |
| `layout/theme.liquid` | Shopify |
| `hugo.toml` / `config.toml` + `layouts/` | Hugo |
| `_config.yml` + `_layouts/` | Jekyll |
| Only `.html` files | Static HTML |

Integration checklist, regardless of stack:

1. Place static files where the stack serves them from root (`public/`, `static/`, `app/` for Next metadata files, `web/`, etc.). Favicons at `/favicon.ico` are fetched by browsers automatically, so keep that path at the site root.
2. Add the `<head>` tags (icons, manifest, `theme-color` for light and dark, Apple tags).
3. Replace existing header/footer logo markup with a theme-aware component or inline SVG. Preserve existing sizing classes, link targets, and `alt`/`aria-label`.
4. Add dark mode handling that matches the project's existing mechanism: Tailwind `dark:` class or `data-theme`, CSS `prefers-color-scheme`, a theme provider, etc. Do not introduce a second theming system.
5. Add Open Graph and Twitter meta tags with **absolute URLs** (relative URLs break social scrapers). Find the site's canonical base URL from config or env; if unknown, leave a clearly marked placeholder and tell the user.
6. Add Schema.org `Organization` JSON-LD with the `logo` property (square or near-square, 112x112 minimum, PNG/SVG/WebP) if the site has none.
7. Respect existing config: base paths (`/subdir/`), CDNs, Content Security Policy (inline SVG and inline `<style>` may need nonces), and service-worker caches.
8. Remove or replace stale icon references (old `favicon.png`, CRA's `logo192.png`, framework default icons) so nothing contradicts the new set.
9. Never overwrite unrelated files. If an existing manifest has other fields (shortcuts, screenshots, lang), merge, do not replace.

### Phase 5 - Theme adaptation (the part most people get wrong)

Pick the technique per place. Details and code are in Appendix B.

| Where | Technique |
|---|---|
| Header/footer logo, single-color logo | Inline SVG or `<svg><use>` with `fill="currentColor"`; set `color` via theme tokens |
| Multi-color logo with 1-2 theme-sensitive colors | Inline SVG with CSS custom properties (`fill: var(--logo-accent)`); override per theme |
| Logo used in `<img>` (cannot be styled by page CSS) | `<picture>` with `prefers-color-scheme` sources, or CSS `mask-image` for flat logos, or the SVG's own internal `@media (prefers-color-scheme: dark)` |
| Favicon (`icon.svg`) | Internal `@media (prefers-color-scheme: dark)` inside the SVG; follows the OS/browser theme, not the site's toggle |
| Site has a manual theme toggle | Prefer inline SVG/CSS variables, because `prefers-color-scheme` does not know about the toggle. For `<img>` logos, render both and toggle visibility with the theme class |
| Logo with dark navy artwork on a dark theme | Provide a light variant or add a subtle light outline/glow; contrast must reach 3:1 |

Rules of thumb: raster `<img>` logos cannot respond to CSS, so use SVG; inline SVG gives full control; SVG inside `<img>`, CSS `background-image`, or a favicon is sandboxed and can only theme itself with its own internal styles.

### Phase 6 - Verification

Do all of these and report results. Do not skip this to save time; broken icons are invisible until a user complains.

- [ ] View every generated image. Compare the symbol shape and colors to the source logo.
- [ ] Render the header logo on light and dark backgrounds, and screenshot if a browser/Playwright is available.
- [ ] Check the 16x16 favicon is recognizable (view it magnified).
- [ ] Confirm each `<link>`/`<meta>` path resolves to a real file (run a script that parses the head and checks files exist).
- [ ] Validate `site.webmanifest` is valid JSON with correct icon sizes and `purpose` values.
- [ ] Confirm `apple-touch-icon.png` has no transparency.
- [ ] Confirm the maskable icon keeps the symbol inside the central 80% (script enforces padding).
- [ ] Confirm OG image is exactly 1200x630 and under about 1 MB.
- [ ] Run the project's build/lint if there is one; fix breakage you caused.
- [ ] Contrast check of the logo against the light and dark theme backgrounds.

### Phase 7 - Report

Finish with a concise summary for the user:

1. What you detected (stack, logo type, colors).
2. Files created and where (a short tree).
3. Files modified (paths only, one line each).
4. Fidelity notes: was the logo traced? How close is it? Any simplification made for the favicon? Any color adjusted for contrast?
5. Anything the user must do: set the production base URL for OG tags, clear favicon cache (browsers cache favicons aggressively; test in a private window), upload `og-image.png` if the platform hosts it, ask their designer for the original vector if the logo was traced.

## Standards cheat sheet

Authoritative sizes and file roles are in Appendix A. The ones you must always get right:

- `favicon.ico` at the site root with 16/32/48 embedded; plus `icon.svg` with `type="image/svg+xml"` for modern browsers.
- Apple touch icon: 180x180, opaque, no pre-rounded corners (iOS rounds them).
- PWA: 192 and 512 `purpose: "any"`, plus a separate 512 `purpose: "maskable"` (do not combine `any maskable` in one file; it produces poor results).
- OG image: 1200x630, PNG or JPG, absolute URL, with `og:image:width`, `og:image:height`, `og:image:alt`.
- `theme-color` meta twice, with `media="(prefers-color-scheme: light|dark)"`.
- Header logo: SVG; PNG fallbacks at 1x/2x/3x only if a legacy context demands them.
- Logo `<img>` must have `width`/`height` attributes (prevents layout shift) and meaningful `alt`; decorative duplicates use `alt=""` and `aria-hidden`.

## Common mistakes to avoid

- Using a PNG/JPG for the header logo and then wondering why dark mode breaks.
- Transparent apple-touch-icon (iOS renders it on black).
- Maskable icon with the symbol touching the edges (Android crops it).
- `fill` hard-coded to black or white so the logo vanishes on one theme.
- `<text>` left in the SVG (falls back to a system font).
- Relative `og:image` URLs.
- Only providing `favicon.ico`, or only an SVG favicon (Safari and older browsers need the ICO/PNG fallback).
- Shipping a 5 MB SVG exported from Illustrator with embedded raster data. If an SVG contains `<image href="data:...">`, it is not a real vector; trace it.
- Overwriting the user's existing manifest, head tags, or theme logic.
- Declaring success without looking at the rendered output.

---

# Appendices

All reference material is included below so this skill works as a single file.

## Appendix A - Asset Matrix

Authoritative list of every asset in a complete web logo system. Sizes are in CSS/pixel units. "Source" tells you what to render it from.


### 1. SVG masters (`brand/svg/`)

| File | Format | Size | Source | Used for | Notes |
|---|---|---|---|---|---|
| `logo.svg` | SVG | viewBox cropped to artwork | Original | Primary full-color logo | Text outlined, no fixed width/height |
| `logo-mono.svg` | SVG | same | Derived | Header/footer that inherit text color | All shapes `fill="currentColor"` |
| `logo-light.svg` | SVG | same | Derived | Light backgrounds | Used with `<picture>` or manual swap |
| `logo-dark.svg` | SVG | same | Derived | Dark backgrounds | Brightened colors or white artwork |
| `logo-horizontal.svg` | SVG | wide ratio | Derived | Navbar | Only if symbol and wordmark both exist |
| `logo-stacked.svg` | SVG | near-square | Derived | Footer, landing, auth pages | Same condition |
| `icon.svg` | SVG | square viewBox (e.g. 0 0 32 32 or 0 0 64 64) | Simplified symbol | Browser favicon | Includes internal `prefers-color-scheme` rules |
| `icon-mono.svg` | SVG | square | Derived | Inline icon use | `currentColor` |
| `safari-pinned-tab.svg` | SVG | square | Derived | Legacy Safari pinned tabs | Single flat color, no gradients, no `<style>` |

### 2. Favicon and device icons (`public/`)

| File | Format | Size | Background | Used for |
|---|---|---|---|---|
| `favicon.ico` | ICO (multi-image) | 16, 32, 48 embedded | Transparent | Root fallback; legacy browsers, crawlers, RSS readers, Windows |
| `icon.svg` | SVG | scalable | Transparent | Modern Chrome, Edge, Firefox (and recent Safari) tab icon |
| `favicon-16x16.png` | PNG | 16x16 | Transparent | Explicit fallback |
| `favicon-32x32.png` | PNG | 32x32 | Transparent | Retina tabs, taskbar |
| `apple-touch-icon.png` | PNG | 180x180 | **Opaque** | iOS/iPadOS home screen, Safari bookmarks. iOS applies its own corner rounding, so supply square corners |
| `icon-192.png` | PNG | 192x192 | Transparent or brand | Android home screen, manifest `any` |
| `icon-512.png` | PNG | 512x512 | Transparent or brand | PWA install, splash, manifest `any` |
| `icon-maskable-512.png` | PNG | 512x512 | **Opaque brand color** | Android adaptive icons, manifest `maskable` |
| `mstile-150x150.png` | PNG | 150x150 | Brand color | Windows pinned tiles (legacy) |

Optional extra Android sizes for older tooling: 48, 72, 96, 144, 384. Modern manifests need only 192 and 512.

### 3. In-page logo rasters (`public/images/` or `brand/png/`)

Only needed for contexts that cannot use SVG (some email clients, legacy CMS fields, PDF generators, third-party widgets).

| File | Format | Size | Notes |
|---|---|---|---|
| `logo@1x.png` | PNG, transparent | height 40 (width by ratio) | Header fallback |
| `logo@2x.png` | PNG, transparent | height 80 | Retina |
| `logo@3x.png` | PNG, transparent | height 120 | High-density phones |
| `logo-dark@1x/2x/3x.png` | PNG, transparent | same | Dark-background variant |
| `logo-2048.png` | PNG, transparent | longest side 2048 | High-res backup for presentations and print proofs |
| `logo-email.png` | PNG | width 400, height by ratio | Display at 200 px wide; email clients often block SVG |

Header display heights: 24-48 px; footer 32-64 px; mobile nav 24-32 px. Always set explicit `width` and `height` attributes.

### 4. Social and search assets

| File | Format | Size | Notes |
|---|---|---|---|
| `og-image.png` | PNG or JPG | **1200x630** (1.91:1) | Facebook, LinkedIn, WhatsApp, Slack, Discord, iMessage. Keep under about 1 MB, absolute URL, keep key content in the central 80% |
| `twitter-image.png` | PNG or JPG | 1200x675 (16:9), or reuse OG | For `summary_large_image` cards |
| `logo-square-512.png` | PNG | 512x512 | Google Knowledge Panel / `Organization.logo`, Slack/Discord app avatars, GitHub org avatar |
| Schema.org logo | PNG/SVG/WebP | min 112x112, ideally square-ish | Referenced in JSON-LD `Organization.logo` |

Social image content: logo centered or left-aligned with generous margin, solid or very simple background in brand color or neutral, optional site name. No tiny text.

### 5. Config files

| File | Purpose |
|---|---|
| `site.webmanifest` (or `manifest.json`) | PWA name, icons, `theme_color`, `background_color`, `display`, `start_url` |
| `browserconfig.xml` | Optional, Windows tiles (legacy; many sites drop it) |
| `head-snippet.html` | Generated block of `<link>` and `<meta>` tags to place in `<head>` |

### Recommended head block

```html
<link rel="icon" href="/favicon.ico" sizes="32x32">
<link rel="icon" href="/icon.svg" type="image/svg+xml">
<link rel="apple-touch-icon" href="/apple-touch-icon.png">
<link rel="manifest" href="/site.webmanifest">
<meta name="theme-color" content="#FFFFFF" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="#0B0B0B" media="(prefers-color-scheme: dark)">
<meta name="color-scheme" content="light dark">
```

### Manifest icons block

```json
"icons": [
  { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any" },
  { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any" },
  { "src": "/icon-maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
]
```

### 6. Padding and safe-zone rules

| Asset | Rule |
|---|---|
| `favicon.ico`, 16/32 PNG | Symbol fills 90-100% of the square. Padding wastes precious pixels |
| `apple-touch-icon` | Opaque brand or neutral background, symbol about 70-80% of the canvas |
| `icon-192/512` (any) | Symbol about 80-90%; transparent is fine |
| `icon-maskable-512` | Symbol inside the central **80% circle** (Android may crop to a circle); in practice scale the symbol bounding box to about 60% of the canvas. Fill the whole canvas with an opaque color |
| `og-image` | Margin of at least 8% on all sides; avoid edges, platforms crop differently |
| `logo-square-512` | 10-15% padding |

### 7. Optional extras (add when the project needs them)

- `apple-touch-startup-image` sets for iOS PWA splash screens (many sizes; usually skip unless it is a serious PWA).
- `screenshots` entries in the manifest for richer install UI.
- `shortcuts` icons (96x96) in the manifest.
- `.well-known` or `humans.txt` branding is unnecessary.
- Windows 11 widgets, Safari mask-icon: skip unless requested.
- Android Play Store listing icon: 512x512 PNG, 32-bit, no transparency, under 1 MB.
- App Store icon: 1024x1024 PNG, no transparency, no rounded corners.
- Chrome Web Store: 128x128 icon, 440x280 small promo tile.
- Favicon for dark/light via two files: `icon-light.svg` / `icon-dark.svg` with `media` attributes on `<link rel="icon">`. Note: browser support for `media` on icon links is inconsistent; prefer the internal SVG media query.

---

## Appendix B - SVG Cleaning and Theming

### 1. Cleaning checklist

- Single root `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 W H">`; remove `width`/`height` (or keep only on standalone files that need an intrinsic size).
- No `<text>`: convert to outlines (Inkscape: Object to Path; Illustrator: Create Outlines; CLI: `inkscape --export-text-to-path`).
- No embedded raster (`<image href="data:image/png...">`). If present, the SVG is not a real vector; trace it.
- Remove editor metadata (`sodipodi:*`, `inkscape:*`, `<metadata>`, Illustrator comments).
- Prefix IDs per file (`logo-grad-1`) when inlining multiple SVGs in one page, otherwise gradient/clip IDs collide.
- Replace `style="fill:#..."` and presentation attributes with classes only if you need theming; otherwise leave as is.
- Keep `fill-rule`/`clip-rule` values; removing them changes how holes render.
- Tight `viewBox`: crop to the artwork bounds, then verify nothing clips (render and view).
- Target under 10 KB for logos, under 3 KB for the favicon icon.

### 2. Technique selection

| Situation | Use |
|---|---|
| One-color logo in header, inline allowed | `currentColor` |
| Multi-color logo, only some colors must flip | CSS custom properties |
| Logo must live in `<img>` | internal `@media (prefers-color-scheme)` or `<picture>` or mask-image |
| Favicon | internal `@media (prefers-color-scheme)` |
| App has a theme toggle plus `<img>` logo | two `<img>`, toggled by theme class, or inline SVG instead |
| Logo's light and dark designs differ substantially | separate `logo-light.svg` / `logo-dark.svg` |

### 3. currentColor

```html
<a href="/" class="brand" aria-label="Acme home">
  <svg viewBox="0 0 120 32" width="120" height="32" fill="currentColor" role="img" aria-hidden="true">
    <path d="..."/>
  </svg>
</a>
```

```css
.brand { color: var(--text); }
```

If the logo has an accent shape, keep it separate: `fill="var(--logo-accent, #0F62FE)"`.

### 4. CSS custom properties

```html
<svg viewBox="0 0 100 100" class="logo">
  <path class="logo-body" d="..."/>
  <circle class="logo-accent" cx="50" cy="50" r="12"/>
</svg>
```

```css
:root {
  --logo-body: #111827;
  --logo-accent: #0F62FE;
}
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) { --logo-body: #F9FAFB; --logo-accent: #4589FF; }
}
:root[data-theme="dark"] { --logo-body: #F9FAFB; --logo-accent: #4589FF; }

.logo-body   { fill: var(--logo-body); }
.logo-accent { fill: var(--logo-accent); }
```

Tailwind variant:

```html
<svg class="h-8 w-auto text-neutral-900 dark:text-white" viewBox="..." fill="currentColor">...</svg>
```

CSS variables work for inline SVG and for `<svg><use href="#id">` (variables inherit into the shadow tree). They do **not** reach into SVG loaded via `<img>` or `background-image`.

### 5. prefers-color-scheme inside the SVG (works in `<img>` and as favicon)

```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 32 32">
  <style>
    .body { fill: #111827; }
    .accent { fill: #0F62FE; }
    @media (prefers-color-scheme: dark) {
      .body { fill: #F9FAFB; }
      .accent { fill: #4589FF; }
    }
  </style>
  <path class="body" d="..."/>
  <circle class="accent" cx="16" cy="16" r="5"/>
</svg>
```

Caveats: this follows the **OS/browser** setting, not a site toggle. The favicon follows the browser UI theme, which can differ from the page theme. Choose colors that look acceptable on both light and dark tab strips when possible (mid-tone brand colors survive both).

### 6. `<picture>` swap

```html
<picture>
  <source srcset="/brand/logo-dark.svg" media="(prefers-color-scheme: dark)">
  <img src="/brand/logo-light.svg" alt="Acme" width="120" height="32">
</picture>
```

Also follows the OS setting only.

### 7. CSS mask-image (flat one-color logos as `<img>` replacement)

```css
.logo-mask {
  display: inline-block;
  width: 120px; height: 32px;
  background-color: currentColor;
  -webkit-mask: url(/brand/logo-mono.svg) no-repeat center / contain;
          mask: url(/brand/logo-mono.svg) no-repeat center / contain;
}
```

Works with any theme because color comes from CSS. Requires a same-origin or CORS-enabled file. Not suitable for multi-color logos. Give it `role="img"` and `aria-label`.

### 8. Manual theme toggles

`prefers-color-scheme` ignores site toggles. Options, in order of preference:

1. Inline SVG plus CSS variables keyed to the site's theme attribute/class.
2. Two `<img>` elements, `.logo-light` and `.logo-dark`, shown/hidden by the theme class (`html.dark .logo-light { display:none }`). Mark the hidden one so it is not announced twice (`display:none` already removes it from the accessibility tree).
3. Swap `src` in JS on theme change (causes a flash; avoid unless needed).

Match the project's existing mechanism (`class="dark"`, `data-theme`, `data-bs-theme`, `.theme-dark`, `useTheme()`), do not invent a new one.

### 9. Fixing contrast

WCAG non-text contrast target: **3:1** against the page background for the logo's essential shapes.

Compute relative luminance, then ratio = (L1 + 0.05) / (L2 + 0.05). Reference backgrounds: `#FFFFFF` (L = 1) and the project's real dark background (default `#0B0B0B`).

Fixes, from least to most invasive:
1. Lighten or darken the problematic color in the dark/light variant by adjusting lightness in HSL/OKLCH while keeping the hue.
2. Switch only the wordmark to white/black; keep the brand-color symbol if it passes.
3. Add a thin contrasting stroke (1-2% of height) around shapes.
4. Place the logo on a rounded-square chip of its own brand background (common for app icons and favicons).
5. Ship dedicated light and dark artwork.

Record every color change and report it to the user; never silently alter brand colors beyond what visibility requires.

### 10. Gradients and multi-color logos

- Gradients survive in favicon SVG but look muddy at 16px; the favicon may flatten to the dominant stop color if that matches the logo's perceived color.
- Use `currentColor` for stops only if a monochrome variant is wanted.
- Safari pinned tab and `logo-mono.svg` must be flat single-color.
- Keep `id`s unique per inlined SVG.

### 11. Accessibility

- Logo link: `<a href="/" aria-label="Acme home">` with the SVG `aria-hidden="true"`, or an `<img alt="Acme">`. Do not do both (double announcement).
- Decorative repeats (footer copy of a logo beside visible text "Acme"): `alt=""` or `aria-hidden="true"`.
- Provide `<title>` inside standalone SVG only when it is meaningful as an image.
- Respect `prefers-reduced-motion` if the logo animates.
- Do not rely on color alone to carry the brand name; the wordmark should be legible.

---

## Appendix C - Framework Integration Recipes

Always detect the stack first, then follow the matching section. Place static assets where the framework serves them from the site root. Replace `https://example.com` with the real canonical origin (read it from config/env; if unknown, leave a marked placeholder and tell the user).


### Shared snippets (referenced below)

### Open Graph + Twitter
```html
<meta property="og:type" content="website">
<meta property="og:site_name" content="Acme">
<meta property="og:title" content="Acme - tagline">
<meta property="og:description" content="Short description">
<meta property="og:url" content="https://example.com/">
<meta property="og:image" content="https://example.com/og-image.png">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:image:alt" content="Acme logo">
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:image" content="https://example.com/og-image.png">
```

### Organization JSON-LD
```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "Acme",
  "url": "https://example.com",
  "logo": "https://example.com/logo-square-512.png"
}
</script>
```

### Icon head block
```html
<link rel="icon" href="/favicon.ico" sizes="32x32">
<link rel="icon" href="/icon.svg" type="image/svg+xml">
<link rel="apple-touch-icon" href="/apple-touch-icon.png">
<link rel="manifest" href="/site.webmanifest">
<meta name="theme-color" content="#FFFFFF" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="#0B0B0B" media="(prefers-color-scheme: dark)">
```

---

### 1. Static HTML
Files in the web root. Paste head block into every page's `<head>` (or the shared include/partial). Inline the SVG logo in the header.

### 2. Next.js (App Router, 13.4+)
Use file-based metadata conventions in `app/`:
- `app/favicon.ico`, `app/icon.svg` (or `icon.png`), `app/apple-icon.png`, `app/opengraph-image.png`, `app/twitter-image.png`, `app/manifest.ts` or `public/site.webmanifest`.
- Next generates the `<link>` tags automatically for these filenames. Do not also hand-write them.
- For extra control use `export const metadata` in `app/layout.tsx`:

```tsx
export const metadata: Metadata = {
  metadataBase: new URL('https://example.com'),
  icons: { icon: [{ url: '/favicon.ico' }, { url: '/icon.svg', type: 'image/svg+xml' }], apple: '/apple-touch-icon.png' },
  manifest: '/site.webmanifest',
  openGraph: { images: [{ url: '/og-image.png', width: 1200, height: 630, alt: 'Acme' }] },
  twitter: { card: 'summary_large_image', images: ['/og-image.png'] },
};
export const viewport: Viewport = {
  themeColor: [
    { media: '(prefers-color-scheme: light)', color: '#FFFFFF' },
    { media: '(prefers-color-scheme: dark)', color: '#0B0B0B' },
  ],
};
```

Logo component (inline SVG, theme via `currentColor`):

```tsx
export function Logo({ className = 'h-8 w-auto' }: { className?: string }) {
  return (
    <svg viewBox="0 0 120 32" fill="currentColor" className={className} role="img" aria-label="Acme">
      <path d="..." />
    </svg>
  );
}
```

For SVGR import, check `next.config` (`@svgr/webpack` or Turbopack rules) before using `import Logo from './logo.svg'`; otherwise inline JSX.

### 3. Next.js (Pages Router)
Put files in `public/`. Add the icon head block, OG tags, and JSON-LD in `pages/_document.tsx` (`<Head>` from `next/document`) or per-page via `next/head`. `viewport`/`theme-color` go in `_document` too.

### 4. React (Vite/CRA) and Remix
- Vite: `public/` served at root. Edit `index.html` `<head>`. Remove default `vite.svg` reference.
- CRA: `public/index.html`; delete `logo192.png`/`logo512.png`/`favicon.ico` defaults it ships, then replace. `%PUBLIC_URL%` prefix on hrefs.
- Remix / React Router framework: `export const links: LinksFunction` in `app/root.tsx` returning `{ rel: 'icon', href: '/favicon.ico' }` etc.; meta via `meta` export.
- Logo as component (inline SVG) or `vite-plugin-svgr` (`import Logo from './logo.svg?react'`) if already configured.

### 5. Vue 3 (Vite) and Nuxt
- Vue + Vite: `public/` + `index.html` head block. Logo as SFC with inline SVG and `currentColor`.
- Nuxt 3: files in `public/`; configure in `nuxt.config.ts`:

```ts
export default defineNuxtConfig({
  app: { head: {
    link: [
      { rel: 'icon', href: '/favicon.ico', sizes: '32x32' },
      { rel: 'icon', href: '/icon.svg', type: 'image/svg+xml' },
      { rel: 'apple-touch-icon', href: '/apple-touch-icon.png' },
      { rel: 'manifest', href: '/site.webmanifest' },
    ],
    meta: [
      { name: 'theme-color', content: '#FFFFFF', media: '(prefers-color-scheme: light)' },
      { name: 'theme-color', content: '#0B0B0B', media: '(prefers-color-scheme: dark)' },
    ],
  } },
});
```
If `@nuxtjs/color-mode` is installed, key logo CSS to `.dark`. If using `@vite-pwa/nuxt` or `@vite-pwa/vite`, put icons in the PWA plugin's manifest config instead of a separate file.

### 6. SvelteKit and Svelte (Vite)
- SvelteKit: `static/` folder; head tags in `src/app.html` (`%sveltekit.assets%/favicon.ico`) or `<svelte:head>` in the root layout. Logo as `Logo.svelte` with inline SVG.
- Dark mode: follow existing mechanism (`mode-watcher`, class on `<html>`).

### 7. Astro
`public/` for files. In the base layout's `<head>`: icon block, OG tags with `Astro.site` for absolute URLs (`new URL('/og-image.png', Astro.site)`). Logo as `Logo.astro` with inline SVG. Set `site` in `astro.config.mjs`.

### 8. Angular
- Files in `src/` assets: add to `angular.json` `assets` array (Angular 17+ uses `public/`). Replace `src/favicon.ico`.
- `src/index.html` head block. Logo as standalone component with inline SVG or `<img>` with `ngSrc` (NgOptimizedImage, supply width/height).
- Use `Meta`/`Title` services for per-route OG tags.

### 9. Laravel (Blade)
Files in `public/`. In the layout (`resources/views/layouts/app.blade.php`) head: use `{{ asset('favicon.ico') }}` for each link. Logo as a Blade component `<x-logo class="h-8" />` with inline SVG. Vite projects: files in `public/` are not hashed; reference via `asset()`. Jetstream/Breeze already ship `application-logo.blade.php`: replace its contents, keep the component name so existing views keep working.

### 10. Django
Put files under `static/` (collected by `collectstatic`) and serve root-level ones (`favicon.ico`, `site.webmanifest`) via a URL pattern (`RedirectView` to `static()`), or place at the project's served root through WhiteNoise's `WHITENOISE_ROOT`. In `base.html`: `{% load static %}` then `<link rel="icon" href="{% static 'favicon.ico' %}">`. Logo via `{% include 'partials/logo.svg' %}` for inline SVG.

### 11. Rails
`public/` for root-level files (`favicon.ico`, manifest). Asset pipeline images via `image_tag`/`asset_path` for fingerprinted ones. Layout `app/views/layouts/application.html.erb` head block: `<%= favicon_link_tag 'favicon.ico' %>`. Inline SVG with `inline_svg` gem if present, otherwise a partial.

### 12. Flask / FastAPI / Express
Serve `static/` (or `public/`) and add explicit routes for `/favicon.ico`, `/site.webmanifest`, and `/apple-touch-icon.png` if static mount is not at root. Express: `app.use(express.static('public'))`. Add head block to the base Jinja/EJS/Handlebars/Pug template. Set long cache headers for hashed assets, shorter (about 1 day) for unhashed favicons.

### 13. WordPress
- Preferred: Appearance > Customize > Site Identity > Site Icon (upload `icon-512.png`, 512x512 minimum, WordPress generates 32/180/192/270). Block themes: Site Editor > Site Logo.
- Programmatic: `add_theme_support('custom-logo')`, and print tags in `wp_head`. Put `og-image` via the active SEO plugin (Yoast/Rank Math) if installed, not in two places.
- SVG uploads are blocked by default; either inline the SVG in a template part or enable safe SVG uploads via a vetted plugin. Do not add an unsanitized upload filter.
- Child themes: modify the child, not the parent.

### 14. Shopify (Liquid)
Theme editor: Settings > Brand assets for logo and favicon (32x32 minimum, Shopify resizes). In code: `layout/theme.liquid` `<head>` with `{{ 'favicon.png' | asset_url }}`; Shopify does not serve arbitrary root files, so `favicon.ico` and the manifest cannot live at `/`; use theme settings plus `asset_url` links. Inline SVG via a snippet `snippets/logo.liquid`.

### 15. Hugo and Jekyll
- Hugo: files in `static/`; head block in `layouts/partials/head.html` (or the theme's override); `{{ "favicon.ico" | absURL }}` for absolute URLs. Logo partial with inline SVG via `readFile`/`safeHTML`, or `resources.Get` for pipelines.
- Jekyll: files in repo root or `assets/`; head block in `_includes/head.html`; use `{{ '/favicon.ico' | relative_url }}`.

### 16. Tailwind and CSS-in-JS notes
- Tailwind: use `text-*` plus `fill-current`, and `dark:` variants matching the configured `darkMode` strategy (`class` vs `media`). Height utilities (`h-8 w-auto`) rather than fixed pixel widths.
- styled-components/Emotion/CSS Modules: expose logo colors as CSS variables on the theme root and consume with `fill: var(--logo-body)`.
- UI libraries (MUI, Chakra, Mantine, shadcn): use the theme's color mode hook to choose `currentColor` or variants; keep the logo inside the provider.

### 18. Service workers, CSP, base paths

- **Base path** (`/app/`, GitHub Pages project sites): prefix icon, manifest, and OG paths; set manifest `start_url` and `scope` accordingly. Favicon `/favicon.ico` fetched by crawlers still wants the domain root, so mention that limitation if the site is served from a subpath.
- **CSP**: inline `<style>` inside SVG favicons is fine (favicon is a separate document), but inline SVG in HTML with `<style>` may require `style-src` allowances or a nonce; prefer classes in the page stylesheet. Check `img-src` allows `data:` only if you used data URIs.
- **Service workers/PWA**: bump the cache version or precache list so the new icons and manifest are picked up; stale caches are a common reason the old icon persists.
- **CDN/cache**: favicons are cached aggressively by browsers. Tell the user to hard-refresh or test in a private window, and consider versioning the filename (`icon.v2.svg`) if the old one is stuck.
- **Monorepos**: apply changes in the correct app package's public directory, not the repo root.

---

## Appendix D - Asset Generator Script (`scripts/generate_assets.py`)

Save this file verbatim, install `cairosvg` and `pillow`, then run it as shown in Phase 3.

```python
#!/usr/bin/env python3
"""
generate_assets.py - Render a complete web logo asset set from SVG masters.

Requires: cairosvg, pillow   (pip install cairosvg pillow --break-system-packages)

Example:
  python generate_assets.py --icon brand/svg/icon.svg --logo brand/svg/logo.svg \
      --out public --name "Acme" --theme-color "#0F62FE"

Outputs (in --out):
  favicon.ico (16/32/48), favicon-16x16.png, favicon-32x32.png
  apple-touch-icon.png (180, opaque), icon-192.png, icon-512.png,
  icon-maskable-512.png, mstile-150x150.png, logo-square-512.png
  logo@1x.png / @2x / @3x (height 40/80/120), logo-2048.png
  og-image.png (1200x630), twitter-image.png (1200x675)
  site.webmanifest, head-snippet.html
"""
import argparse
import io
import json
import os
import sys

try:
    import cairosvg
    from PIL import Image
except ImportError:
    sys.exit("Missing dependencies. Run: pip install cairosvg pillow --break-system-packages")


def hex_to_rgba(value, alpha=255):
    value = value.lstrip("#")
    if len(value) == 3:
        value = "".join(c * 2 for c in value)
    r, g, b = (int(value[i:i + 2], 16) for i in (0, 2, 4))
    return (r, g, b, alpha)


def luminance(rgb):
    def ch(c):
        c = c / 255
        return c / 12.92 if c <= 0.03928 else ((c + 0.055) / 1.055) ** 2.4
    r, g, b = rgb[:3]
    return 0.2126 * ch(r) + 0.7152 * ch(g) + 0.0722 * ch(b)


def render_svg(svg_path, width=None, height=None, scale=None):
    """Render an SVG to a transparent RGBA PIL image."""
    kwargs = {"url": svg_path}
    if width:
        kwargs["output_width"] = int(width)
    if height:
        kwargs["output_height"] = int(height)
    if scale:
        kwargs["scale"] = scale
    png = cairosvg.svg2png(**kwargs)
    return Image.open(io.BytesIO(png)).convert("RGBA")


def fit_on_canvas(symbol, canvas_size, fill_ratio, background=None):
    """Place an RGBA image centered on a square/rect canvas, scaled so its
    larger side occupies fill_ratio of the smaller canvas side."""
    cw, ch = canvas_size if isinstance(canvas_size, tuple) else (canvas_size, canvas_size)
    target = int(min(cw, ch) * fill_ratio)
    w, h = symbol.size
    scale = target / max(w, h)
    new = symbol.resize((max(1, round(w * scale)), max(1, round(h * scale))), Image.LANCZOS)
    canvas = Image.new("RGBA", (cw, ch), background or (0, 0, 0, 0))
    canvas.alpha_composite(new, ((cw - new.width) // 2, (ch - new.height) // 2))
    return canvas


def render_square(svg_path, size, fill_ratio=1.0, background=None):
    # render large then downsample for better antialiasing at tiny sizes
    big = render_svg(svg_path, width=size * 4, height=size * 4)
    # Trim nothing: icon.svg viewBox is expected to be tightly composed already
    canvas = fit_on_canvas(big, size * 4, fill_ratio, background)
    return canvas.resize((size, size), Image.LANCZOS)


def save_png(img, path):
    img.save(path, format="PNG", optimize=True)
    print(f"  wrote {os.path.relpath(path)}  {img.size[0]}x{img.size[1]}")


def main():
    p = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    p.add_argument("--icon", required=True, help="Square symbol SVG (simplified for small sizes)")
    p.add_argument("--logo", help="Full logo SVG (symbol + wordmark, or same as icon). Defaults to --icon")
    p.add_argument("--logo-dark", help="Optional dark-background logo SVG for dark PNG variants")
    p.add_argument("--out", default="public", help="Output directory")
    p.add_argument("--name", default="My Site", help="Site/app name")
    p.add_argument("--short-name", help="Short name for manifest (default: --name)")
    p.add_argument("--description", default="", help="Manifest description")
    p.add_argument("--theme-color", default="#0F62FE", help="Brand color used for tiles/maskable bg")
    p.add_argument("--bg-light", default="#FFFFFF", help="Light page background")
    p.add_argument("--bg-dark", default="#0B0B0B", help="Dark page background")
    p.add_argument("--icon-bg", help="Opaque background for apple-touch/maskable (default: bg-light)")
    p.add_argument("--og-bg", help="Background for OG/Twitter image (default: bg-light)")
    p.add_argument("--base-url", default="", help="Absolute origin, e.g. https://example.com (for OG tags)")
    p.add_argument("--start-url", default="/", help="Manifest start_url")
    p.add_argument("--no-manifest", action="store_true", help="Skip site.webmanifest")
    args = p.parse_args()

    logo = args.logo or args.icon
    out = args.out
    os.makedirs(out, exist_ok=True)
    short = args.short_name or args.name
    icon_bg = hex_to_rgba(args.icon_bg or args.bg_light)
    og_bg = hex_to_rgba(args.og_bg or args.bg_light)
    print("Generating assets...")

    # --- Favicons -----------------------------------------------------------
    f16 = render_square(args.icon, 16, 1.0)
    f32 = render_square(args.icon, 32, 1.0)
    f48 = render_square(args.icon, 48, 1.0)
    save_png(f16, os.path.join(out, "favicon-16x16.png"))
    save_png(f32, os.path.join(out, "favicon-32x32.png"))
    # ICO with embedded 16/32/48
    base = render_square(args.icon, 256, 1.0)
    base.save(os.path.join(out, "favicon.ico"), format="ICO", sizes=[(16, 16), (32, 32), (48, 48)])
    print("  wrote favicon.ico  (16,32,48)")

    # --- Apple touch icon (opaque) -----------------------------------------
    save_png(render_square(args.icon, 180, 0.78, icon_bg).convert("RGB").convert("RGBA"),
             os.path.join(out, "apple-touch-icon.png"))

    # --- Android / PWA ------------------------------------------------------
    save_png(render_square(args.icon, 192, 0.86), os.path.join(out, "icon-192.png"))
    save_png(render_square(args.icon, 512, 0.86), os.path.join(out, "icon-512.png"))
    # Maskable: opaque bg, symbol inside central safe zone (~60% bounding box)
    mask_bg = hex_to_rgba(args.icon_bg or args.theme_color)
    mask_icon = render_square(args.icon, 512, 0.60, mask_bg)
    save_png(mask_icon, os.path.join(out, "icon-maskable-512.png"))
    save_png(render_square(args.icon, 150, 0.70, hex_to_rgba(args.theme_color)),
             os.path.join(out, "mstile-150x150.png"))
    save_png(render_square(args.icon, 512, 0.80), os.path.join(out, "logo-square-512.png"))

    # --- In-page logo rasters ----------------------------------------------
    probe = render_svg(logo, width=800)
    ratio = probe.width / probe.height
    for mult, name in ((1, "logo@1x.png"), (2, "logo@2x.png"), (3, "logo@3x.png")):
        h = 40 * mult
        img = render_svg(logo, width=round(h * ratio * 2), height=h * 2).resize(
            (round(h * ratio), h), Image.LANCZOS)
        save_png(img, os.path.join(out, name))
    big = render_svg(logo, width=2048) if ratio >= 1 else render_svg(logo, height=2048)
    save_png(big, os.path.join(out, "logo-2048.png"))
    if args.logo_dark:
        for mult in (1, 2, 3):
            h = 40 * mult
            img = render_svg(args.logo_dark, width=round(h * ratio * 2), height=h * 2).resize(
                (round(h * ratio), h), Image.LANCZOS)
            save_png(img, os.path.join(out, f"logo-dark@{mult}x.png"))

    # --- Social cards -------------------------------------------------------
    def social(size, name):
        w, h = size
        canvas = Image.new("RGBA", size, og_bg)
        mark = render_svg(logo, width=w * 2) if ratio >= 1 else render_svg(logo, height=h * 2)
        # logo occupies up to 60% width and 45% height
        s = min((w * 0.60) / mark.width, (h * 0.45) / mark.height)
        mark = mark.resize((round(mark.width * s), round(mark.height * s)), Image.LANCZOS)
        canvas.alpha_composite(mark, ((w - mark.width) // 2, (h - mark.height) // 2))
        save_png(canvas.convert("RGB").convert("RGBA"), os.path.join(out, name))
    social((1200, 630), "og-image.png")
    social((1200, 675), "twitter-image.png")

    # --- Manifest -----------------------------------------------------------
    if not args.no_manifest:
        manifest = {
            "name": args.name,
            "short_name": short,
            "description": args.description,
            "start_url": args.start_url,
            "display": "standalone",
            "background_color": args.bg_light,
            "theme_color": args.theme_color,
            "icons": [
                {"src": "/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any"},
                {"src": "/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any"},
                {"src": "/icon-maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable"},
            ],
        }
        if not args.description:
            manifest.pop("description")
        with open(os.path.join(out, "site.webmanifest"), "w") as fh:
            json.dump(manifest, fh, indent=2)
            fh.write("\n")
        print("  wrote site.webmanifest")

    # --- Head snippet -------------------------------------------------------
    base_url = args.base_url.rstrip("/")
    og_url = f"{base_url}/og-image.png" if base_url else "https://REPLACE-WITH-YOUR-DOMAIN/og-image.png"
    tw_url = f"{base_url}/twitter-image.png" if base_url else "https://REPLACE-WITH-YOUR-DOMAIN/twitter-image.png"
    snippet = f"""<!-- Icons -->
<link rel="icon" href="/favicon.ico" sizes="32x32">
<link rel="icon" href="/icon.svg" type="image/svg+xml">
<link rel="apple-touch-icon" href="/apple-touch-icon.png">
<link rel="manifest" href="/site.webmanifest">
<meta name="theme-color" content="{args.bg_light}" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="{args.bg_dark}" media="(prefers-color-scheme: dark)">
<meta name="color-scheme" content="light dark">

<!-- Open Graph / Twitter -->
<meta property="og:type" content="website">
<meta property="og:site_name" content="{args.name}">
<meta property="og:image" content="{og_url}">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:image:alt" content="{args.name} logo">
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:image" content="{tw_url}">
"""
    with open(os.path.join(out, "head-snippet.html"), "w") as fh:
        fh.write(snippet)
    print("  wrote head-snippet.html")

    # --- Contrast report ----------------------------------------------------
    sample = render_square(args.icon, 64, 1.0)
    raw = sample.tobytes()
    px = [tuple(raw[i:i + 4]) for i in range(0, len(raw), 4) if raw[i + 3] > 200]
    if px:
        avg = tuple(sum(c[i] for c in px) // len(px) for i in range(3))
        def ratio_vs(bg_hex):
            l1, l2 = luminance(avg), luminance(hex_to_rgba(bg_hex))
            hi, lo = max(l1, l2), min(l1, l2)
            return (hi + 0.05) / (lo + 0.05)
        print("\nContrast of average icon color vs backgrounds (want >= 3.0):")
        print(f"  vs light {args.bg_light}: {ratio_vs(args.bg_light):.2f}")
        print(f"  vs dark  {args.bg_dark}: {ratio_vs(args.bg_dark):.2f}")
    print("\nDone. Remember to view the outputs and copy icon.svg to the output directory.")


if __name__ == "__main__":
    main()
```

## Appendix E - Verification Script (`scripts/verify_assets.py`)

Save and run after integration: `python verify_assets.py --dir public --html path/to/layout-or-index.html`

```python
#!/usr/bin/env python3
"""
verify_assets.py - Check a logo asset directory (and optional HTML head file).

Usage:
  python verify_assets.py --dir public [--html public/index.html]

Checks: required files exist, PNG dimensions, apple-touch-icon opacity, OG size,
manifest validity, ICO sizes, SVG hygiene, and that href/src in the HTML resolve.
Exit code 1 if any check fails.
"""
import argparse
import json
import os
import re
import sys

try:
    from PIL import Image
except ImportError:
    sys.exit("Missing pillow. Run: pip install pillow --break-system-packages")

failures = 0


def ok(msg):
    print(f"  [ok]   {msg}")


def bad(msg):
    global failures
    failures += 1
    print(f"  [FAIL] {msg}")


def warn(msg):
    print(f"  [warn] {msg}")


EXPECTED = {
    "favicon-16x16.png": (16, 16),
    "favicon-32x32.png": (32, 32),
    "apple-touch-icon.png": (180, 180),
    "icon-192.png": (192, 192),
    "icon-512.png": (512, 512),
    "icon-maskable-512.png": (512, 512),
    "og-image.png": (1200, 630),
}


def check_dir(d):
    print(f"Checking {d}")
    for name, size in EXPECTED.items():
        path = os.path.join(d, name)
        if not os.path.exists(path):
            bad(f"{name} missing")
            continue
        with Image.open(path) as im:
            if im.size != size:
                bad(f"{name} is {im.size}, expected {size}")
            else:
                ok(f"{name} {size[0]}x{size[1]}")
            if name in ("apple-touch-icon.png", "icon-maskable-512.png", "og-image.png"):
                rgba = im.convert("RGBA")
                lo = rgba.getchannel("A").getextrema()[0]
                if lo < 255:
                    bad(f"{name} has transparency (should be opaque)")
                else:
                    ok(f"{name} is opaque")
            if name == "og-image.png":
                kb = os.path.getsize(path) / 1024
                (ok if kb < 1024 else warn)(f"og-image.png is {kb:.0f} KB (aim for < 1024 KB)")

    ico = os.path.join(d, "favicon.ico")
    if os.path.exists(ico):
        with Image.open(ico) as im:
            sizes = sorted(im.info.get("sizes", {im.size}))
            if {(16, 16), (32, 32)}.issubset(set(sizes)):
                ok(f"favicon.ico contains {sizes}")
            else:
                bad(f"favicon.ico sizes {sizes}; need at least 16 and 32")
    else:
        bad("favicon.ico missing")

    svg = os.path.join(d, "icon.svg")
    if os.path.exists(svg):
        text = open(svg, encoding="utf-8").read()
        if "viewBox" not in text:
            bad("icon.svg has no viewBox")
        else:
            ok("icon.svg has viewBox")
        if "<text" in text:
            bad("icon.svg contains <text>; outline it")
        if "data:image" in text:
            bad("icon.svg embeds a raster image; not a true vector")
        if "prefers-color-scheme" in text:
            ok("icon.svg has dark-mode rules")
        else:
            warn("icon.svg has no prefers-color-scheme rule (fine if colors work on both themes)")
    else:
        warn("icon.svg not found in output dir")

    mf = os.path.join(d, "site.webmanifest")
    if os.path.exists(mf):
        try:
            data = json.load(open(mf))
            icons = data.get("icons", [])
            purposes = {(i.get("sizes"), i.get("purpose", "any")) for i in icons}
            if ("192x192", "any") in purposes and ("512x512", "any") in purposes:
                ok("manifest has 192/512 'any' icons")
            else:
                bad("manifest missing 192x192 or 512x512 'any' icon")
            if ("512x512", "maskable") in purposes:
                ok("manifest has maskable icon")
            else:
                warn("manifest has no maskable icon")
            if any("any maskable" in (i.get("purpose") or "") for i in icons):
                warn("avoid combining 'any maskable' in one icon")
            for i in icons:
                rel = i["src"].lstrip("/")
                if not os.path.exists(os.path.join(d, rel)):
                    bad(f"manifest icon not found: {i['src']}")
            for k in ("name", "theme_color", "background_color"):
                if k not in data:
                    warn(f"manifest missing '{k}'")
        except json.JSONDecodeError as e:
            bad(f"manifest invalid JSON: {e}")
    else:
        warn("site.webmanifest not found")


def check_html(html_path, root):
    print(f"Checking HTML references in {html_path}")
    text = open(html_path, encoding="utf-8", errors="ignore").read()
    refs = re.findall(r'<link[^>]+rel="(?:[^"]*icon[^"]*|manifest)"[^>]*href="([^"]+)"', text)
    refs += re.findall(r'<link[^>]+href="([^"]+)"[^>]*rel="(?:[^"]*icon[^"]*|manifest)"', text)
    if not refs:
        bad("no icon/manifest <link> tags found")
    for r in sorted(set(refs)):
        if r.startswith(("http://", "https://", "data:")):
            ok(f"external/data reference {r[:60]}")
        elif os.path.exists(os.path.join(root, r.lstrip("/"))):
            ok(f"{r} resolves")
        else:
            bad(f"{r} does not resolve under {root}")
    for og in re.findall(r'property="og:image"\s+content="([^"]+)"', text):
        if og.startswith("http"):
            ok("og:image is absolute")
        else:
            bad(f"og:image must be an absolute URL, got {og}")
    if 'media="(prefers-color-scheme: dark)"' in text and 'name="theme-color"' in text:
        ok("theme-color has dark variant")
    else:
        warn("no dark theme-color meta")


if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("--dir", required=True)
    ap.add_argument("--html")
    a = ap.parse_args()
    check_dir(a.dir)
    if a.html:
        check_html(a.html, a.dir)
    print("\nResult:", "PASS" if failures == 0 else f"{failures} FAILURE(S)")
    sys.exit(1 if failures else 0)
```
