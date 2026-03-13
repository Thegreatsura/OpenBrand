# OpenBrand

Try it out at https://openbrand.sh

Extract brand assets (logos, colors, backdrops, brand name) from any website URL.

## As an npm package

```bash
bun add openbrand
```

```typescript
import { extractBrandAssets } from "openbrand";

const brand = await extractBrandAssets("https://stripe.com");
// brand.brand_name → "Stripe"
// brand.logos → LogoAsset[]
// brand.colors → ColorAsset[]
// brand.backdrop_images → BackdropAsset[]
```

Server-side only (requires Node.js/Bun for cheerio and sharp).

## Self-hosting the web app

```bash
git clone https://github.com/ethanjyx/openbrand.git
cd openbrand
bun install
bun dev
```

No environment variables required. Open http://localhost:3000.

## What it extracts

- **Logos** — favicons, apple-touch-icons, header/nav logos, inline SVGs (with dimension probing)
- **Brand colors** — from theme-color meta tags, manifest.json, and dominant colors from logo imagery
- **Backdrop images** — og:image, CSS backgrounds, hero/banner images
- **Brand name** — from og:site_name, application-name, logo alt text, page title

## How it works

OpenBrand fetches the page HTML (falling back to [Jina](https://jina.ai/) for JS-heavy sites) and runs a multi-stage extraction pipeline:

**1. Candidate collection** — Gathers all image sources from the page: `<link>` favicons and apple-touch-icons, every `<img>` tag, and inline SVGs found inside header/nav logo containers. Each candidate is tagged with metadata: its DOM location (header, footer, body), whether its src/alt/class contains "logo", whether the domain name appears in the URL or alt text, and whether it sits inside a hero/banner section.

**2. Dimension probing** — Fires off parallel `probe-image-size` requests to get width, height, and aspect ratio for every candidate without downloading the full image.

**3. Two-pass classification** — Candidates are classified into logos or backdrops:
- *Pass 1 (high-confidence)*: Favicons and apple-touch-icons are always logos. Inline SVGs from header/nav are always logos. `<img>` tags in header/nav with a logo hint or domain match (≤500px) are logos. Images in hero/banner sections (≥400px) are backdrops.
- *Pass 2 (fallback)*: Only runs if pass 1 found zero logos. Looks for `<img>` tags in footer/body that have both a logo hint and domain match (≤500px).

This two-pass approach prevents third-party logos (e.g. customer logos on a landing page) from being misidentified as the site's own logo.

**4. Color extraction** — Collects theme colors from `<meta name="theme-color">`, `msapplication-TileColor`, and `manifest.json`. Then extracts dominant colors from logo images by downscaling to 16x16 with Sharp, weighting pixels by saturation so chromatic brand colors rank above grays/whites. Deduplicates by RGB distance.

**5. Backdrop extraction** — Separately collects `og:image` meta tags, CSS `background-image` URLs from `<style>` blocks and inline styles.

**6. Brand name** — Tries `og:site_name`, `application-name`, logo alt text (cleaned of "logo"/"icon" suffixes), then the shortest segment of `<title>` (split on `|`, `-`, `—`), falling back to the capitalized domain name.

## Tech stack

Next.js, React, TypeScript, Cheerio, Sharp, Tailwind CSS

## License

[MIT](LICENSE)
