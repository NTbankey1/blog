# Airspace Hugo API & Component Reference

This guide documents the public surface area of the Airspace Hugo theme, including its automation scripts, frontend JavaScript, Hugo shortcodes, layout partials, widgets, and SCSS mixins. Each section summarizes responsibilities, configuration knobs, and includes usage examples so you can confidently extend or integrate the theme.

## Conventions

- File paths appear in backticks, for example `assets/js/script.js`.
- Hugo-specific snippets use Go Template syntax.
- Configuration examples reference Hugo's `config/_default/*.toml` files unless noted.

---

## Node Automation Scripts (`themes/airspace-hugo/scripts`)

The project ships two Node-based utilities to convert between the standalone theme package and a self-contained project. Both scripts execute immediately when required (`node <script>.js`) and expose helper functions you can reuse in custom tooling.

### `projectSetup.js`

| Function | Signature | Description | Parameters | Returns |
| --- | --- | --- | --- | --- |
| `getFolderName` | `getFolderName(rootfolder)` | Reads `exampleSite/config/_default/hugo.toml` to determine the selected theme name. | `rootfolder`: absolute path to the project root. | `string \| null` theme folder name. |
| `deleteFolder` | `deleteFolder(folderPath)` | Recursively removes a directory if it exists. | `folderPath`: absolute path. | `void` |
| `createNewfolder` | `createNewfolder(rootfolder, folderName)` | Ensures a nested directory exists (recursively), returning its path. | `rootfolder`: base path; `folderName`: folder to create. | Absolute path of the created directory. |
| `iterateFilesAndFolders` | `iterateFilesAndFolders(rootFolder, { destinationRoot })` | Walks `rootFolder` recursively, mirroring directories and moving files into `destinationRoot`. | `rootFolder`: source path; `destinationRoot`: destination path. | `void` |
| `setupProject` | `setupProject()` | Entry point. If no `themes/` directory exists, migrates `layouts`, `assets`, and `static` into `themes/<themeName>/` and promotes the example site content into the project root. | None | `void` |

**Usage example**

```bash
node themes/airspace-hugo/scripts/projectSetup.js
```

Call this script after cloning the theme repository into a project to rehydrate the theme subdirectory structure.

### `themeSetup.js`

| Function | Signature | Description | Parameters | Returns |
| --- | --- | --- | --- | --- |
| `createNewfolder` | `createNewfolder(rootfolder, folderName)` | Same implementation as in `projectSetup.js`. | See above. | Absolute path of the created directory. |
| `deleteFolder` | `deleteFolder(folderPath)` | Deletes a directory tree if present. | `folderPath`: absolute path. | `void` |
| `getFolderName` | `getFolderName(rootfolder)` | Fetches the configured theme name from `exampleSite/config/_default/hugo.toml`. | `rootfolder`: absolute path. | `string \| null` |
| `iterateFilesAndFolders` | `iterateFilesAndFolders(rootFolder, { destinationRoot })` | Moves files and folders recursively. | See above. | `void` |
| `setupTheme` | `setupTheme()` | Entry point. When the example site is absent, scaffolds `exampleSite/`, migrates project-level content/config/assets back under it, and expands the theme stored in `themes/<themeName>/` into the project root. | None | `void` |

**Usage example**

```bash
node themes/airspace-hugo/scripts/themeSetup.js
```

Run this when preparing the repository for theme distribution (restoring the canonical `exampleSite` structure).

---

## Frontend JavaScript

### `assets/js/script.js`

The main client script enhances UI components after the DOM loads.

- **Passive touch listeners**: Overrides jQuery's `touchstart` and `touchmove` setup to use passive listeners unless the event namespace contains `noPreventDefault`.
- **Preloader dismissal**: On `window.load`, elements with the `.preloader` class fade out immediately. Pair with the `preloader` partial.
- **Shuffle filtering**: When `.shuffle-wrapper` exists, instantiates `Shuffle` with `.shuffle-item` children and listens to `input[name="shuffle-filter"]` changes to filter items.
- **Sliders**: Initializes Slick sliders on `.portfolio-single-slider`, `.clients-logo`, and `.testimonial-slider` with autoplay every 2 seconds.
- **Counter animation**: Defines `counter()` to animate elements with class `.count` from their current text to the `data-count` attribute once scrolled into view.
- **Cloaked email reveal**: Transforms spans rendered by the `cloak_email` shortcode/partial into clickable `mailto:` links by reversing the obfuscated data attributes.
- **Map bootstrapping**: Calls the global `map()` initializer (defined in `assets/plugins/google-map/gmap.js`) via jQuery when available.

**Implementation hint**: Ensure relevant vendor bundles (`shuffle`, `slick`, `jquery`) are listed under `params.plugins.js` in `hugo.toml` so the assets load before this script runs.

**Example HTML structure**

```html
<div class="shuffle-wrapper">
  <div class="shuffle-item" data-groups='["branding"]'>...</div>
</div>
<label>
  <input type="radio" name="shuffle-filter" value="branding" checked>
  Branding
</label>
```

### `assets/plugins/google-map/gmap.js`

Global `map()` renders a styled Google Map when a `#map` element is present.

- Reads required coordinates and marker metadata from `#map` data attributes.
- Applies grayscale custom styling and disables most map controls.
- Creates a single pin using the supplied `data-marker` image and `data-marker-name` title.
- Registers `initialize()` on Google Maps' `load` event.

**Usage steps**

1. Enable Google Maps loading in `config/_default/params.toml`:

   ```toml
   [gmap]
   enable = true
   gmap_api = "YOUR_API_KEY"
   ```

2. Add the map container to the Contact page:

   ```html
   <div
     id="map"
     data-latitude="40.7128"
     data-longitude="-74.0060"
     data-marker="/images/marker.png"
     data-marker-name="Headquarters">
   </div>
   ```

---

## Hugo Shortcodes (`themes/airspace-hugo/layouts/shortcodes`)

### `button`

Renders a styled CTA button. Two positional parameters: label (`.Get 0`) and URL (`.Get 1`). External URLs open in a new tab with `rel="noopener"`.

```gotemplate
{{< button "Start a Project" "/contact" >}}
```

### `cloak_email`

Obfuscates an email address to defeat scrapers. Accepts a single positional parameter (`EMAIL`). Pairs with the frontend script that restores a clickable link.

```gotemplate
Need help? {{< cloak_email "hello@example.com" >}}
```

### `codepen`

Embeds a CodePen by slug hash (positional parameter). Requires the CodePen embed script to be listed under `params.plugins.js`.

```gotemplate
{{< codepen "abCDef" >}}
```

### `collapse`

Builds a single accordion item using Bootstrap 5 classes.

- `.Get 0`: Accordion title (also used to derive DOM ids).
- `.Inner`: Markdown-rendered body content.

```gotemplate
{{< collapse "What is Airspace?" >}}
Airspace is a responsive Hugo theme built for agencies.
{{< /collapse >}}
```

### `date_l10n`

Formats a date string according to the current language.

- `.Get 0`: Input date (parsable by `time.Format`).
- `.Get 1` (optional): Hugo time format layout (defaults to `:date_long`).

```gotemplate
{{< date_l10n "2025-03-21" >}}
{{< date_l10n "2025-03-21" ":date_medium" >}}
```

### `image`

Feature-rich responsive image helper supporting CDN links, page bundle resources, and asset pipeline transforms.

| Parameter | Required | Description |
| --- | --- | --- |
| `src` | Yes | Relative path (page resource or assets) or absolute URL. |
| `alt` | Yes | Accessible alternative text. |
| `caption` | No | Figure caption (wraps output in `<figure>`). |
| `position` | No | One of `center`, `left`, `right`, `float-left`, `float-right`. |
| `class` | No | Additional CSS classes. |
| `height`, `width` | No | Target dimensions, e.g. `400x` or `400px`. |
| `command` | No | Image processing command: `fit`, `fill`, `resize` (defaults to `resize`). |
| `option` | No | Extra image processing options (e.g. `Lanczos`). |

Automatic behaviors:

- Local assets convert to WebP with a fallback format (unless SVG/GIF).
- CDN images bypass processing and render directly.
- Missing assets emit an inline error message.

```gotemplate
{{< image src="images/hero.jpg" alt="Design team" caption="Collaborative work" command="fill" width="800" height="500" >}}
```

### `tabs` & `tab`

Wrap multiple tab panes. Use `tabs` as the container and nest `tab` shortcodes for each pane. The theme's scripts populate the tab headers client-side.

```gotemplate
{{< tabs >}}
  {{< tab "HTML" >}}
  <div>Hello</div>
  {{< /tab >}}
  {{< tab "CSS" >}}
  .hello { color: #f00; }
  {{< /tab >}}
{{< /tabs >}}
```

---

## Layout Partials (`themes/airspace-hugo/layouts/partials`)

### `blog-sidebar`

Renders a widget column using the list defined at `site.Params.widgets.sidebar`. Each entry should be the filename (without extension) of a widget partial inside `partials/widgets`.

```toml
[params.widgets]
  sidebar = ["recent_posts", "taxonomy_category", "taxonomy_tags"]
```

Include in templates via:

```gotemplate
{{ partial "blog-sidebar.html" . }}
```

### `cloak_email`

Partial equivalent of the shortcode for when you need to cloak an address within a template or partial. Pass the raw email string.

```gotemplate
{{ partial "cloak_email.html" "jobs@example.com" }}
```

### `cta`

Outputs the global call-to-action section defined on the homepage:

- Looks for front matter under `content/<lang>/_index.md` -> `params.cta`.
- Expects `bg_image`, optional `title`, `content`, and a `button` object with `enable`, `label`, `link`.

Sample front matter:

```yaml
cta:
  bg_image: "images/call-to-action-bg.jpg"
  title: "Ready to start?"
  content: "Let's build something great together."
  button:
    enable: true
    label: "Contact Us"
    link: "/contact"
```

### `footer`

Displays footer navigation and copyright text.

- Menu items sourced from `menus.<lang>.toml` (`footer` menu).
- Copyright text reads `site.Params.copyright` and supports Markdown.

### `head`

Sets up meta tags, favicons, structured social metadata, and multiple analytics integrations based on site params.

Key parameters:

- `site.Params.description`, `author`, `image`.
- `site.Params.favicon` (processed via Hugo pipes when located under `assets/`).
- `site.Params.variables.color_primary`, `body_color` (used for browser theme colors).
- Analytics toggles: `site.Params.google_tag_manager`, `site.Params.matomo`, `site.Params.baidu`, `site.Params.plausible`, `site.Params.counter`, `site.Params.contact.form.use_recaptcha` with `site.Params.recaptcha_site_key`.

### `header`

Builds the responsive top navigation.

- `site.Params.navbar_fixed`: if `true`, adds `sticky-top`.
- Menu hierarchy driven by `site.Menus.main`; supports nested dropdowns and external links.
- Includes language switcher when the current page has translations.
- Embeds the logo partial (`logo.html`).

### `logo`

Responsible for rendering either an image logo or fallback text.

- Configure via `site.Params.logo` (path under `assets/`) and optional `logo_width` (e.g. `150px`).
- Converts raster formats to WebP and sets a fallback, preserving SVGs and GIFs.
- If no image is provided, falls back to `site.Params.logo_text` or the site title.

### `page-title`

Hero/banner section for inner pages.

- Requires `.Params.bg_image` in page front matter.
- Uses `.Title` and optional `.Params.description` for heading and subheading.

### `preloader`

Injects a preloader overlay when enabled.

- Toggle via `site.Params.preloader.enable`.
- Specify the asset path with `site.Params.preloader.preloader` (processed through Hugo pipes, with WebP + fallback generation for non-GIF formats).

### `script`

Aggregates JavaScript assets and optional consent banner.

- Google Maps loading gated by `site.Params.gmap.enable` and only on the Contact page.
- External scripts defined in `site.Params.plugins.js` render as `<script src="...">`; local scripts are concatenated and fingerprinted with the theme's main bundle.
- Cookie consent banner controlled by `site.Params.cookies.enable`, with `content`, `button`, and `expire_days` fields.
- Loads Google Web Fonts using `site.Params.variables.font_primary` and `font_secondary`.

Example configuration:

```toml
[[params.plugins.js]]
link = "plugins/jquery/jquery.min.js"

[[params.plugins.js]]
link = "plugins/slick/slick.min.js"

[params.cookies]
enable = true
content = "We use cookies to improve experience."
button = "Got it"
expire_days = 30
```

### `service`

Outputs service cards sourced from the `/service` page.

- Expects front matter under `params.service` with `title`, `description`, and a `service_item` array (`icon`, `name`, `content`).

### `style`

Builds the CSS pipeline, combining external plugin styles and the theme's SCSS.

- External styles: declare under `site.Params.plugins.css`.
- Local styles: compiles `assets/scss/style.scss` after executing it as a template (giving access to site params for dynamic CSS).
- The concatenated result is inlined with Subresource Integrity (`fingerprint "sha512"`).

### Widgets (`partials/widgets`)

| Partial | Purpose | Data sources |
| --- | --- | --- |
| `widget_area` | Iterates through widget names provided via `.Widgets` and renders each widget partial with the supplied scope. | `dict` with `Widgets` (string array) and `Scope` (page context). |
| `recent_posts` | Shows the four most recent `post`-type pages with optional thumbnails. | `where site.Pages "Type" "post"` |
| `taxonomy_category` | Lists all categories, highlighting the current term when browsing taxonomy pages. | `site.Taxonomies.categories` |
| `taxonomy_tags` | Lists all tags, highlighting the current term when applicable. | `site.Taxonomies.tags` |

To add a custom widget, create `themes/airspace-hugo/layouts/partials/widgets/<name>.html` and reference its name in `params.widgets.sidebar`.

---

## SCSS Mixins (`assets/scss/_mixins.scss`)

These mixins standardize CSS transforms and transitions across browsers. Import `_mixins.scss` into your custom stylesheet (e.g. `assets/scss/custom.scss`) and apply them within other rules.

| Mixin | Signature | Description |
| --- | --- | --- |
| `transition` | `@include transition($what: all, $time: 0.2s, $how: ease-in-out);` | Shorthand for applying prefixed transition properties. |
| `transition-multi` | `@include transition-multi($x...);` | Accepts a vararg list of transitions (e.g. `opacity 0.3s ease, transform 0.3s ease`). |
| `transform` | `@include transform($transforms);` | Sets prefixed `transform` declarations. |
| `rotate` | `@include rotate(45);` | Rotate helper built on `transform`. Accepts degrees without the `deg` suffix. |
| `scale` | `@include scale(1.1);` | Scales an element uniformly. |
| `translate` | `@include translate(10px, 0);` | Applies `translate(X, Y)`. |
| `skew` | `@include skew(15, 0);` | Applies skew in degrees on X/Y axes. |
| `transform-origin` | `@include transform-origin(center center);` | Sets prefixed transform origin. |

**Example**

```scss
.cta-button {
  @include transition(background-color 0.3s ease);

  &:hover {
    @include translate(0, -2px);
  }
}
```

---

## Additional Notes

- Most partials assume multilingual content; ensure you configure `languages.toml` and `i18n/*.toml` files to match your locales.
- When adding new vendor assets, place local files under `assets/plugins/...` and append their paths to `params.plugins.css` or `params.plugins.js` so the Hugo pipes include them in the compiled bundles.
- Frontend features that rely on jQuery or other plugins require the scripts to load before `js/script.js`; verify order in your `params.plugins.js` array.

Use this document as the canonical reference when extending the theme or onboarding new contributors to Airspace.

