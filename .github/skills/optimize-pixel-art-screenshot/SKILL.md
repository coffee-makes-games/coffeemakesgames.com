---
name: optimize-pixel-art-screenshot
description: >-
    Skill for optimizing pixel-art game screenshots (typically iOS simulator captures
    or similar high-resolution PNG exports) into web-ready WebP files. Use when the
    user wants to add a screenshot to a website, reduce file size of pixel-art PNGs,
    or convert simulator exports for web delivery. Preserves crisp pixel-art look
    while dramatically reducing file size.
user-invocable: true
---

# Optimize pixel-art screenshot for the web

Convert a high-resolution pixel-art PNG (typically a phone-resolution simulator screenshot, e.g. ~1320×2868) into a web-ready WebP that is ~25× smaller while remaining visually identical at the display size.

## When to use

- User has dropped a raw `.png` screenshot into the repo (often `assets/<game>/screenshot-N.png`) and wants it shown on the website.
- User mentions screenshot file size, retina sharpness, or asks how to use a screenshot.
- User asks to convert PNG → WebP for any pixel-art / low-color-palette image.

Do NOT use for:
- Photographic images (different workflow — use lossy WebP at q80).
- Vector art (those should be SVG, not converted).
- Images already < 100KB (skip optimization, just wire them up).

## Why these choices

- **WebP over PNG.** Lossy WebP at q90 reduces a typical 2.5MB pixel-art PNG to ~100KB with no visible degradation, because pixel art uses a small color palette that compresses extremely well.
- **Resize to 720px wide.** Display slot on a typical site is ~280–360px wide. 720px gives 2× retina sharpness without shipping wasteful pixels. Adjust if the design slot is significantly larger or smaller.
- **Quality 90 (lossy), not lossless.** For pixel art with a 16-color palette, q90 lossy is visually indistinguishable from lossless and ~7× smaller. Lossless WebP only wins when the user explicitly demands it (e.g., for archival).
- **`image-rendering: pixelated`** in CSS is required so browsers don't blur the pixel art when scaling.

## Prerequisites

Check that `cwebp` is installed:
```bash
which cwebp
```

If not, install via Homebrew on macOS:
```bash
brew install webp
```

`sips` is built into macOS and used for resizing.

## Procedure

### Step 1 — Inspect the source

Confirm the source is pixel art (not photo) and check current dimensions / size:
```bash
sips -g pixelWidth -g pixelHeight <path-to-png> | tail -3
ls -lh <path-to-png>
```

If the file is < 100KB and dimensions are already close to the display size, skip optimization — just wire it up as PNG.

### Step 2 — Resize to retina display width

Default target: **720px wide** (2× of a ~360px display slot). Adjust based on the actual design.

```bash
sips --resampleWidth 720 <path-to-source.png> --out /tmp/resized.png
```

`sips --resampleWidth` preserves aspect ratio automatically.

### Step 3 — Convert to WebP at q90

```bash
cwebp -q 90 /tmp/resized.png -o <path-to-output.webp>
```

Verify the resulting size is reasonable (typically 50–200KB for a phone-screenshot-sized pixel-art frame):
```bash
ls -lh <path-to-source.png> <path-to-output.webp>
```

If the WebP is unexpectedly large (> 500KB), the source may not be pixel art — re-check before continuing.

### Step 4 — Wire into the page

Update the `<img src="...">` to point at the new `.webp` file. Update the `alt` text to describe the actual content (no longer "placeholder").

Ensure the corresponding CSS selector includes:
```css
image-rendering: pixelated;
image-rendering: crisp-edges;
```
(Both lines — `pixelated` is the modern keyword, `crisp-edges` is the legacy fallback. Set the latter second so it does not override.)

### Step 5 — Decide what to do with the source PNG

The original high-resolution PNG is the source of truth. Options to discuss with the user:

- **Delete** if they have the simulator capture they can re-export from anytime.
- **Move to `assets/<game>/_source/`** and add to `.gitignore` to keep originals out of the deployed site.
- **Keep alongside `.webp`** — wastes some repo size but the PNG is not served by the site (no `<img>` references it).

Default to asking the user; do not delete sources without explicit consent.

## Batch operation

When the user has multiple screenshots to process (e.g., they drop `screenshot-1.png` through `screenshot-6.png`), run the same procedure in a single bash chain rather than per-file separate calls. Example:

```bash
cd /path/to/assets/<game> && \
for n in 1 2 3 4 5 6; do
  if [ -f "screenshot-$n.png" ]; then
    sips --resampleWidth 720 "screenshot-$n.png" --out "/tmp/r-$n.png" >/dev/null 2>&1
    cwebp -q 90 "/tmp/r-$n.png" -o "screenshot-$n.webp" 2>&1 | tail -1
  fi
done && \
ls -lh screenshot-*.webp | awk '{print $5"\t"$9}'
```

Then update each `<img>` reference in the HTML in a single batched edit pass.

## Quality tuning

If the user reports the WebP looks soft or has artifacts:

1. Try `-q 95` (still lossy, ~1.5× larger, virtually indistinguishable from lossless).
2. If still unhappy, fall back to `cwebp -lossless` (typically 5–7× larger than q90 but bit-perfect).

If the user wants smaller files and the source is mostly flat-color UI:

1. Try `-q 80` (often imperceptible for low-palette art, ~30% smaller than q90).
2. Do not go below q75 for pixel art — the palette starts to dither visibly.

## Anti-patterns

- ❌ Do NOT convert pixel art to JPEG (block artifacts on pixel edges look terrible).
- ❌ Do NOT convert pixel art to SVG (raster → vector is impossible without redrawing).
- ❌ Do NOT resize using bicubic / smooth scaling tools without `pixelated` CSS — the result blurs.
- ❌ Do NOT skip the `image-rendering: pixelated` CSS — even a perfect WebP will look soft if the browser is allowed to smooth-scale it.
- ❌ Do NOT write a script file to the repo for this — use bash chains. The user explicitly prefers skills over scripts.
