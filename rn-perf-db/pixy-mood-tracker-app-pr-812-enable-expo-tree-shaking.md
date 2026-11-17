# perf(build): enable Expo tree shaking for release bundles
- **Repository:** mrzmyr/pixy-mood-tracker-app
- **PR Link:** <https://github.com/mrzmyr/pixy-mood-tracker-app/pull/812>
- **Merged Date:** 2026-10-18
- **Category:** Bundle Size

## Problem
Release bundles shipped all exports of every module. Measured with `expo export --platform ios` on the production variant: minified JS 13.27 MB, Hermes bytecode 14.16 MB, 6,531 JS modules in the source map.

## Solution
Turn on Expo's experimental tree shaking for release builds only.
- `eas.json`: `preview` and `production` profiles set `EXPO_UNSTABLE_METRO_OPTIMIZE_GRAPH=1` and `EXPO_UNSTABLE_TREE_SHAKING=1`.
- `metro.config.js`: defaults both to `1` unless the variant is `development`, so local release builds match; `??=` keeps explicit values, so `0` opts out.
- Result reported: minified JS 13.27 → 11.00 MB (-17%), Hermes bytecode 14.16 → 11.86 MB (-16%), gzip bytecode 4.58 → 4.05 MB, modules 6,531 → 4,124.
- Risks noted: Expo marks it experimental and it changes the production module graph; a preview build smoke test on device is needed. An earlier measurement showed Hermes gzip size growing, so watch store download size.

## Code Pattern (Before vs. After)
```json
// Before (eas.json)
"production": {
  "env": {
    "EXPO_PUBLIC_APP_VARIANT": "production"
  }
}

// After
"production": {
  "env": {
    "EXPO_PUBLIC_APP_VARIANT": "production",
    "EXPO_UNSTABLE_METRO_OPTIMIZE_GRAPH": "1",
    "EXPO_UNSTABLE_TREE_SHAKING": "1"
  }
}
```
```js
// After (metro.config.js, top of file)
if (process.env.EXPO_PUBLIC_APP_VARIANT !== "development") {
  process.env.EXPO_UNSTABLE_METRO_OPTIMIZE_GRAPH ??= "1";
  process.env.EXPO_UNSTABLE_TREE_SHAKING ??= "1";
}
```

## Takeaway
On Expo SDK 54+, tree shaking can be enabled per build profile; compare bundles with and without it and smoke-test a release build before shipping.
