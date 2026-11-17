# fix: use polyfill-force to improve startup time
- **Repository:** mattermost/mattermost-mobile
- **PR Link:** <https://github.com/mattermost/mattermost-mobile/pull/7904>
- **Merged Date:** 2024-04-22
- **Category:** Startup

## Problem
On a Samsung Galaxy S9 the app took over 7 s from splash screen to server screen. Logging between `index.ts` and the server screen showed the largest cost was loading the `@formatjs` Intl polyfills: each `require('@formatjs/*/polyfill')` runs `shouldPolyfill()`, which in some cases took over 2000 ms. Release-build measurements from the PR (ms, normal vs. forced): Galaxy S9 5523 to 200, Pixel 6 2069 to 100, Galaxy S22 2815 to 97, iPhone X 2495 to 109, iPhone 13 Pro 3119 to 123.

## Solution
- Hermes always needs these Intl polyfills here, so the capability check is wasted work.
- Switched every `@formatjs` polyfill import in `index.ts` from `/polyfill` to `/polyfill-force`, which installs the polyfill without calling `shouldPolyfill()`.
- Per the PR, unsupported locales default to `en`, so skipping the check is safe. A dynamic-loading approach was tried first and dropped once `shouldPolyfill()` was identified as the cost.

## Code Pattern (Before vs. After)
```ts
// Before
if (global.HermesInternal) {
    // Polyfills required to use Intl with Hermes engine
    require('@formatjs/intl-getcanonicallocales/polyfill');
    require('@formatjs/intl-locale/polyfill');
    require('@formatjs/intl-pluralrules/polyfill');
    require('@formatjs/intl-numberformat/polyfill');
    require('@formatjs/intl-datetimeformat/polyfill');
    require('@formatjs/intl-datetimeformat/add-golden-tz');
    require('@formatjs/intl-listformat/polyfill');
}

// After
if (global.HermesInternal) {
    // Polyfills required to use Intl with Hermes engine
    require('@formatjs/intl-getcanonicallocales/polyfill-force');
    require('@formatjs/intl-locale/polyfill-force');
    require('@formatjs/intl-pluralrules/polyfill-force');
    require('@formatjs/intl-numberformat/polyfill-force');
    require('@formatjs/intl-datetimeformat/polyfill-force');
    require('@formatjs/intl-datetimeformat/add-golden-tz');
    require('@formatjs/intl-listformat/polyfill-force');
}
```

## Takeaway
If you always install an Intl polyfill on Hermes, import the `polyfill-force` entry to skip the feature-detection cost at startup.
