# Responsive Images in Astro

Examples of using Astro's `<Image />` component (from `astro:assets`) to produce optimized, responsive images from **local** and **remote** sources.

```sh
npm install
npm run dev       # http://localhost:4321
npm run build     # catches errors the dev server may not show
npm run preview   # view the site created by the 'build' command
```

## Pages

| Page | What it shows |
| --- | --- |
| `/single-image/` | One local image imported from `src/assets/` |
| `/group-of-images/` | A folder of local images loaded with `import.meta.glob` |
| `/single-remote-image/` | One remote image (URL) |
| `/group-of-remote-images/` | An array of remote image URLs from www.nps.gov|

## Key idea: Astro needs width and height

To generate `srcset` and prevent layout shift, `<Image />` must know each image's original dimensions.

- **Local images** — Astro reads them at build time. An import gives you `ImageMetadata` (`src`, `width`, `height`, `format`).
- **Remote images** — Astro can't see the file until it fetches it, so you add `inferSize` (or pass `width`/`height` yourself).

## Local images

Images must live in `src/` (e.g. `src/assets/`), **not** `public/`, to be optimized.

### Single image

```astro
---
import { Image } from "astro:assets";
import glacier from "../assets/glacier-national-park.jpg";
---
<Image src={glacier} alt="Glacier National Park" layout="constrained" />
```

### A folder of images

```astro
---
import { Image } from "astro:assets";

const imageImports = import.meta.glob<{ default: ImageMetadata }>(
  "../assets/birds/*.{jpeg,jpg,png,webp}",
  { eager: true },
);
const images = Object.values(imageImports).map((mod) => mod.default);
---
{images.map((image) => (
  <Image src={image} alt="Bird" layout="constrained"
    sizes="(max-width: 600px) 100vw, (max-width: 1280px) 50vw, 25vw" />
))}
```

Note the type is `{ default: ImageMetadata }` — each glob result is a module whose `default` export is the metadata.

## Remote images

### 1. Authorize the domain

In `astro.config.mjs`, so Astro will optimize (resize + convert) the images instead of passing them through untouched:

```js
import { defineConfig } from "astro/config";

export default defineConfig({
  image: { domains: ["www.nps.gov"] },
});
```

### 2. Use `inferSize`

```astro
---
import { Image } from "astro:assets";
const imageUrls = [
  "https://www.nps.gov/common/uploads/structured_data/3C7D5920-1DD8-B71B-0B83F012ED802CEA.jpg",
  // ...
];
---
{imageUrls.map((src) => (
  <Image src={src} inferSize alt="National park" layout="constrained"
    sizes="(max-width: 768px) 100vw, (max-width: 1280px) 33vw, 25vw" />
))}
```

`inferSize` fetches the start of each file at build time to read its dimensions. No extra libraries (`image-size`, `node-fetch`) are needed.

### Need the dimensions as data?

For things like a lightbox, CSS `aspect-ratio`, or sorting by orientation, use `inferRemoteSize`:

```js
import { inferRemoteSize } from "astro:assets";
const { width, height } = await inferRemoteSize(url);
```

## Responsive options

| Prop | Purpose |
| --- | --- |
| `layout` | `"constrained"` (scale down, never up), `"full-width"`, `"fixed"`, or `"none"`.  |
| `sizes` | Tells the browser how wide the image will display at each breakpoint. Override when the auto value doesn't match your CSS (e.g. a grid). |
| `widths` | Explicit list of widths to generate. Good to have, but Astro picks reasonable defaults. |
| `format` | Output format. Defaults to `webp`. |
| `alt` | Required. Describe the image, or use `alt=""` for purely decorative ones. |

## Gotchas

- **`inferSize` does nothing on local images** — they already have dimensions.
- **Test with `npm run build`.** The dev server only renders pages you visit; the build renders all of them.

## Relevant Astro Docs

- [Images guide](https://docs.astro.build/en/guides/images/)
- [`<Image />` reference](https://docs.astro.build/en/reference/modules/astro-assets/)
