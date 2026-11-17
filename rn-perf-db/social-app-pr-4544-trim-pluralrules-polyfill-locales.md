# Fix Android startup perf regression
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/4544>
- **Merged Date:** 2024-06-18
- **Category:** Startup

## Problem
The `@formatjs/intl-pluralrules` polyfill was slow at startup on Android, because it processed every locale the package supports. The author did not investigate why and filed an upstream issue. Measured time to first paint over 5 runs each: roughly 2675-2928 ms before, 1901-1929 ms after.

## Solution
- Patch (patch-package) `supported-locales.generated.js` in `@formatjs/intl-pluralrules` so `supportedLocales` lists only the 16 locales the app actually ships.
- Add a custom ESLint rule, `keep-i18n-patch-in-sync`, that errors when code imports a pluralrules locale-data file that is not in the patched list, so new locales cannot be added without updating the patch.
- Caveat: the author did not verify what this part of the polyfill does internally, but assumes excluding unused locales is safe; language switching was tested.

## Code Pattern (Before vs. After)
```js
// Before: node_modules/@formatjs/intl-pluralrules/supported-locales.generated.js
// (package default: generated list of every CLDR locale)

// After (patches/@formatjs+intl-pluralrules+5.2.10.patch)
// PATCHED FOR ANDROID PERF. KEEP IN SYNC WITH i18n.ts
exports.supportedLocales = [
  'ca', 'de', 'en', 'es', 'fi', 'fr', 'ga', 'hi',
  'id', 'it', 'ja', 'ko', 'pt', 'tr', 'uk', 'zh'
];
```
```js
// eslint/keep-i18n-patch-in-sync.js
if (!supportedLocales.includes(code)) {
  context.report({
    node,
    message: 'Edit .patches/@formatjs+intl-pluralrules+XXX.patch to include ' + code,
  })
}
```

## Takeaway
Profile Intl polyfills at startup; restrict them to the locales you ship and guard the restriction with a lint rule.
