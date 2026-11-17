# feat: handle lazy load date fns locales
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/64619>
- **Merged Date:** 2025-06-24
- **Category:** Bundle Size (+ Startup)

## Problem
`DateUtils` statically imported ten date-fns locales (de, en-GB, es, fr, it, ja, nl, pl, pt-BR, zh-CN) and picked one in a `setLocale` switch, so all of them were bundled and evaluated at startup although a user needs only one. The PR does not report measurements.

## Solution
Extend the lazy translation loading into a locale store that loads the app translations and the matching date-fns locale together, via `Promise.all` of two dynamic imports per locale, and caches both. After the locale loads, the store calls date-fns `setDefaultOptions({locale})` with the cached locale. `DateUtils` no longer imports any locale, and its `setLocale` switch is removed. As in the translations PR, dynamic imports are written out per locale with string literals because Metro cannot resolve computed paths.

## Code Pattern (Before vs. After)
```ts
// Before: DateUtils.ts
import {de} from 'date-fns/locale/de';
import {es} from 'date-fns/locale/es';
// ...eight more locale imports

function setLocale(localeString: Locale | undefined) {
    switch (localeString) {
        case CONST.LOCALES.EN:
            setDefaultOptions({locale: enGB});
            break;
        case CONST.LOCALES.ES:
            setDefaultOptions({locale: es});
            break;
        // ...
    }
}

// After: IntlStore.ts
[LOCALES.ES]: () =>
    this.cache.has(LOCALES.ES)
        ? Promise.all([Promise.resolve(), Promise.resolve()])
        : Promise.all([
              import('./es').then((module) => {
                  this.cache.set(LOCALES.ES, flattenObject(extractModuleDefaultExport(module)));
              }),
              import('date-fns/locale/es').then((module) => {
                  this.dateUtilsCache.set(LOCALES.ES, module.es);
              }),
          ]),
```
