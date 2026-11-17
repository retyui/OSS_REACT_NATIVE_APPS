# [LIVE-22324] Update usage of Intl polyfills for performance on Android
- **Repository:** LedgerHQ/ledger-live
- **PR Link:** <https://github.com/LedgerHQ/ledger-live/pull/12318>
- **Merged Date:** 2025-10-16
- **Category:** Startup

## Problem
The formatjs Intl polyfills (`intl-pluralrules`, `intl-relativetimeformat`) can slow down Android startup, as documented by formatjs. The app imported the polyfills plus locale data for 9 languages statically at boot, even though only one language is used per session. On a Pixel 7, four launches went from about 5.4-5.5 s before to about 4.2-4.5 s after (e.g. 5467 ms -> 4362 ms).

## Solution
- Switch to the `polyfill-force` entry points (the optimized variant intended for devices without native support) for `intl-locale`, `intl-pluralrules` and `intl-relativetimeformat`.
- Remove the static `locale-data/*` imports and add a `localeLoader` that `require`s only the locale data for the active language, with an `en` fallback and a loaded-language cache.
- The default language is loaded in `polyfill.ts` at startup; `LocaleProvider` loads data again via an effect when the language changes.

## Code Pattern (Before vs. After)
```ts
// Before
import "@formatjs/intl-locale/polyfill";
import "@formatjs/intl-pluralrules/polyfill";
import "@formatjs/intl-pluralrules/locale-data/en";
import "@formatjs/intl-pluralrules/locale-data/fr";
import "@formatjs/intl-pluralrules/locale-data/es";
// ... ru, zh, de, tr, ja, ko, then the same list for relativetimeformat

// After
import "@formatjs/intl-locale/polyfill-force";
import "@formatjs/intl-pluralrules/polyfill-force";
import "@formatjs/intl-relativetimeformat/polyfill-force";

import { getDefaultLanguageLocale } from "./languages";
import { loadLocaleData } from "./utils/localeLoader";
loadLocaleData(getDefaultLanguageLocale());
```

```ts
// localeLoader.ts (new)
export const loadLocaleData = (language: string): void => {
  if (loadedLanguages.has(language)) return;
  switch (language) {
    case "fr":
      require("@formatjs/intl-pluralrules/locale-data/fr");
      require("@formatjs/intl-relativetimeformat/locale-data/fr");
      break;
    // ...
  }
  loadedLanguages.add(language);
};
```

## Takeaway
Do not statically import every locale's polyfill data at startup; load only the active language lazily with `require`.
