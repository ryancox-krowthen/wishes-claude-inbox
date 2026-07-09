# Wishes UI Asset Pack

This package contains the baseline CSS design system and starter asset pack for the Wishes web portal.

## Contents

- `src/styles/index.css` — single CSS entry point
- `src/styles/variables.css` — canonical design tokens
- `src/styles/components/` — reusable component CSS
- `src/styles/pages/` — page-level CSS
- `examples/asset-review.html` — complete example asset review page
- `assets/svg/realms/` — realm badge SVG placeholders
- `assets/svg/qualities/` — quality badge SVG placeholders
- `assets/svg/ui/` — logo and frame placeholders
- `assets/reference/` — generated reference palette images
- `design-tokens.json` — machine-readable token export

## Recommended install

Copy `src/styles` into the Wishes web portal and import:

```ts
import "./styles/index.css";
```

Or include in HTML:

```html
<link rel="stylesheet" href="../src/styles/index.css" />
```

## Rule

Use variables from `variables.css` for colors, spacing, radius, shadows, and layout dimensions. Do not hardcode these values in page or component CSS.
