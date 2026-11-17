# chore: optimize PNG images with lossy compression (-47.7 MB)
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/28915>
- **Merged Date:** 2026-04-17
- **Category:** App Size (+ Images)

## Problem
Sentry Size Analysis flagged image optimization as the top actionable insight for reducing the app binary size. Several bundled PNG illustrations were multi-megabyte (for example `stacked-cards.png` 8.1M -> 1.4M, `predict-marketing.png` 6.2M -> 1.7M). The PR reports a total saving of 47.7 MB across 154 images.

## Solution
- Run lossy palette reduction with `pngquant` (quality 65-85, metadata stripped) over `app/`, modifying only files where savings exceed 4 KB (matching Sentry's threshold).
- Add `scripts/optimize-images.sh` (supports `--dry-run`, a target directory and quality flags; also handles JPEG via mozjpeg) so the step can be repeated.
- Image owners were asked to check visual quality; the PR states no perceptible loss at mobile DPIs and that images can be re-exported from source if any look degraded.

## Code Pattern (Before vs. After)
```bash
# Before
# PNG assets committed as exported (several 2-8 MB illustrations)

# After
./scripts/optimize-images.sh --dry-run   # preview savings
./scripts/optimize-images.sh             # apply

# core of the script
pngquant --quality="65-85" --strip --force --output "$tmp" "$file"
```

## Takeaway
Run an automated lossy image pass (pngquant/mozjpeg) with a minimum-savings threshold, and keep the script in the repo for future assets.
