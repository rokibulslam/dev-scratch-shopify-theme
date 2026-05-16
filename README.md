# dev-scratch-shopify-theme (Horizon)

Notes and reference docs for the **Shopify Horizon (OS 2.0)** theme architecture in this repo.

---

## Theme architecture (high-level)

Horizon is an **Online Store 2.0** theme. Rendering is composed from these layers:

- **Layout (wiring + globals)**: `layout/theme.liquid`
  - Loads global CSS/JS and defines the page shell.
  - Renders global header/footer groups.
  - Defines the `<main id="MainContent">` area where the current template content renders.

- **Templates (page composition)**: `templates/*.json`
  - Define which **sections** render for each route (product, collection, index, etc.) and their order.
  - Contain section settings and block settings (data/state), not heavy logic.

- **Sections (page-level building blocks)**: `sections/*.liquid`
  - Liquid markup + `{% schema %}` for Theme Editor settings/blocks.
  - Often captures block markup and delegates layout to snippets.

- **Blocks (reusable units inside sections)**: `blocks/*.liquid`
  - Block types used within sections (example: product media, product details, buttons, accordion rows).

- **Snippets (shared UI + glue)**: `snippets/*.liquid`
  - Shared rendering helpers (buttons, cards, product parts, etc.).
  - Also contains global “glue” snippets like stylesheets/scripts/variables.

- **Assets (CSS/JS)**: `assets/*`
  - Global CSS files (example: `assets/base.css`)
  - Module scripts loaded via importmap (example: `assets/section-renderer.js`)

Key wiring in `layout/theme.liquid`:

```liquid
{%- render 'stylesheets' -%}
{%- render 'fonts' -%}
{%- render 'scripts' -%}
{%- render 'theme-styles-variables' -%}
{%- render 'color-schemes' -%}

<div id="header-group">
  {% sections 'header-group' %}
</div>

<main id="MainContent">
  {{ content_for_layout }}
</main>

<footer>
  {% sections 'footer-group' %}
</footer>
```

---

## Product page architecture

### Template composition (`templates/product.json`)

`templates/product.json` defines the Product page as **sections + order**. In this repo it includes:

- **`product-information`** (main product media + details area)
- **`product-recommendations`** (related/complementary products)

### Main product section (`sections/product-information.liquid`)

This section is the “controller”:

- Captures the two core blocks:
  - **Media gallery**: block type `_product-media-gallery`
  - **Details panel**: block type `_product-details`
- Optionally renders **sticky add-to-cart**
- Delegates the layout/grid to:
  - `snippets/product-information-content.liquid`

Flow:

`templates/product.json` → `sections/product-information.liquid` → `snippets/product-information-content.liquid` → blocks/snippets

### Layout grid (`snippets/product-information-content.liquid`)

This snippet:

- Chooses render order based on `desktop_media_position` (media left/right)
- Builds the grid wrapper and classes like:
  - `product-information--media-left|right`
  - `product-information--media-none`
- Outputs:
  - media container
  - details container
  - then any additional blocks

### Media gallery implementation

- Entry block: `blocks/_product-media-gallery.liquid`
- Core renderer: `snippets/product-media-gallery-content.liquid`
  - Builds slides and/or a desktop grid
  - Renders `<media-gallery ... data-presentation="grid|carousel">`
  - Handles zoom dialog, aspect ratios, constraints, thumbnails
- Styles helper: `snippets/product-media-gallery-content-styles.liquid`

### Details panel implementation

- Entry block: `blocks/_product-details.liquid`
- Renders its children via the `group` system (layout settings like width, sticky, padding, alignment).
- Typical child blocks configured in `templates/product.json` include:
  - title / text
  - price
  - variant picker
  - buy buttons
  - description
  - accordion / inventory / custom properties (depending on configuration)

### Recommendations (`sections/product-recommendations.liquid`)

- Renders `<product-recommendations ...>` and hydrates results client-side.
- Supports grid or carousel.
- Uses `data-hydration-key` to support section hydration updates.

---

## CSS architecture (global vs section vs user-added)

### Global CSS (loaded on every page)

`layout/theme.liquid` renders `snippets/stylesheets.liquid`, which loads:

```liquid
{{ 'overflow-list.css' | asset_url | preload_tag: as: 'style' }}
{{ 'base.css' | asset_url | stylesheet_tag: preload: true }}
```

So the main global stylesheet is:

- `assets/base.css`

### Design tokens / variables (global)

- `snippets/theme-styles-variables.liquid` generates a large `:root { ... }` token set:
  - typography families/weights/sizes
  - spacing scales
  - border radii, z-index layers, etc.

### Section-scoped CSS (most UI styling)

Horizon heavily uses `{% stylesheet %}` blocks inside:

- `sections/*.liquid`
- `blocks/*.liquid`
- (some) `snippets/*.liquid`

This pattern keeps CSS close to the component/section that owns it.

### Dynamic section updates (Section Rendering API)

- `assets/section-renderer.js` fetches and morphs section HTML.
- It can also inject/replace section stylesheet blocks by extracting:
  - `style[data-section-stylesheet]`

### “User-end CSS” (merchant custom CSS/HTML)

The merchant-facing escape hatch is:

- `sections/custom-liquid.liquid`

It renders `{{ section.settings.custom_liquid }}` directly, so merchants can inject:

- `<style>...</style>` CSS
- custom HTML
- external `<link>` tags (if they choose)

This is **not automatically scoped** by the theme; it’s raw output.

---

## Special files (architecture anchors)

- **Layout / entrypoints**
  - `layout/theme.liquid`: page shell + global includes + header/footer groups
  - `snippets/stylesheets.liquid`: global CSS entrypoint (loads `assets/base.css`)
  - `snippets/scripts.liquid`: importmap + modulepreload + module scripts
  - `snippets/theme-styles-variables.liquid`: global CSS variables / tokens

- **Header / footer group system**
  - `sections/header-group.json` + `sections/footer-group.json`: group composition for global header/footer
  - `sections/header.liquid` + `sections/footer.liquid`: group section implementations (often include `{% stylesheet %}`)

- **Product page**
  - `templates/product.json`: product page composition
  - `sections/product-information.liquid`: product controller wrapper
  - `snippets/product-information-content.liquid`: product layout grid
  - `blocks/_product-media-gallery.liquid` + `snippets/product-media-gallery-content*.liquid`: gallery + zoom + styles
  - `blocks/_product-details.liquid`: details panel + children via group system
  - `sections/product-recommendations.liquid`: recommendations element + hydration

- **Dynamic rendering**
  - `assets/section-renderer.js`: Section Rendering API fetch + DOM morph + optional stylesheet injection

