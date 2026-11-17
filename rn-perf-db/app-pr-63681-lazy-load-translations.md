# [NoQA] Implemented lazy-load translations
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/63681>
- **Merged Date:** 2025-06-16
- **Category:** Bundle Size (+ Startup)

## Problem
`translations.ts` statically imported and flattened every language file (`en` and `es`) at module load, so all locales were parsed and held in memory at startup regardless of the user's language. The PR gives no measurements; its test plan is to verify only one locale is downloaded on web.

## Solution
Introduce a `TranslationStore` that loads a locale on demand with a dynamic `import('./en')` / `import('./es')`, flattens it, and caches it. Loading sets an `ARE_TRANSLATIONS_LOADING` Onyx flag, and `Expensify.tsx` only initialises once `preferredLocale` is available, so the splash screen waits for the translations. `translateLocal` call sites in components were switched to the context `translate` so they re-render when the locale loads. Switching language downloads the new locale first and then applies it. Loaders use string-literal imports because Metro requires them.

## Code Pattern (Before vs. After)
```ts
// Before: src/languages/translations.ts
import en from './en';
import es from './es';

export default {
    en: flattenObject(en),
    es: flattenObject(es),
};

// After: src/languages/TranslationStore.ts
private static loaders = {
    [CONST.LOCALES.EN]: () => {
        if (this.localeCache.has(CONST.LOCALES.EN)) {
            return Promise.resolve();
        }
        return import('./en').then((module) => {
            this.localeCache.set(CONST.LOCALES.EN, flattenObject(module.default));
        });
    },
    // ...same for ES
};
```

```tsx
// Before
const shouldInit = isNavigationReady && hasAttemptedToOpenPublicRoom;

// After
const shouldInit = isNavigationReady && hasAttemptedToOpenPublicRoom && !!preferredLocale;
```
