# [NoQA][Audit][Implementation] - Remove Intl Polyfills which are supported by Hermes on native platforms
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/38228>
- **Merged Date:** 2024-03-18
- **Category:** Startup

## Problem
Native startup ran several `@formatjs` Intl polyfills (getCanonicalLocales, PluralRules, NumberFormat, DateTimeFormat) that Hermes now implements natively. The PR's profiler trace (release build, real Android device, heavy account) shows almost ~500 ms spent polyfilling APIs before the change.

## Solution
Drop the polyfills that Hermes already supports from the native entry point and keep only `intl-locale` and the custom ListFormat polyfill, which Hermes does not provide. After the change the profiler no longer shows any polyfill trace (the PR says the time is negligible).

## Code Pattern (Before vs. After)
```ts
// Before
import polyfillDateTimeFormat from './polyfillDateTimeFormat';
import polyfillListFormat from './polyfillListFormat';
import polyfillNumberFormat from './polyfillNumberFormat';

const intlPolyfill: IntlPolyfill = () => {
    // Native devices require extra polyfills
    require('@formatjs/intl-getcanonicallocales/polyfill');
    require('@formatjs/intl-locale/polyfill');
    require('@formatjs/intl-pluralrules/polyfill');
    polyfillNumberFormat();
    polyfillDateTimeFormat();
    polyfillListFormat();
};

// After
import polyfillListFormat from './polyfillListFormat';

const intlPolyfill: IntlPolyfill = () => {
    // Native devices require extra polyfills which are
    // not yet implemented in hermes.
    // see support: https://hermesengine.dev/docs/intl/
    require('@formatjs/intl-locale/polyfill');

    polyfillListFormat();
};
```

## Takeaway
Audit startup polyfills against what your Hermes version ships; each one runs synchronously before first render.
