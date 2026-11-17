# ref: lazy-load translations
- **Repository:** BlueWallet/BlueWallet
- **PR Link:** <https://github.com/BlueWallet/BlueWallet/pull/8559>
- **Merged Date:** 2026-05-19
- **Category:** Memory

## Problem
`loc/index.ts` built the `Localization` instance with all 53 language JSON files (`en` plus about 50 others) required eagerly, so every dictionary was parsed and held in the heap even though one language is active. The PR measured this on an Android release APK (Pixel 3a API 34 emulator, Hermes, 10 cold launches per variant). Median PSS was 300,536 KB patched vs 304,251 KB baseline, which is -3.63 MB. Startup time showed no measurable change (+21 ms median, within one stdev). Bundle and APK size are unchanged because Metro still inlines every JSON.

## Solution
- `Localization` is created with only `{ en: enJson }`.
- Every other language is a thunk in a typed `languageLoaders` map (`() => require('./ru.json')`), and `LangCode` is derived from `AvailableLanguages` so any drift becomes a TypeScript error.
- `applyLanguage(lang)` runs the loader on demand, caches the parsed dictionary in `parsedLanguages`, and calls `loc.setContent({ en, [lang] })` followed by `setLanguage`, with `en` kept as the fallback.
- Caveat noted in the code: on a cold launch with a non-English saved preference, `loc` starts as `{en}` until the async init resolves, so English can briefly render.

## Code Pattern (Before vs. After)
```ts
// Before
const loc: ILocalization = new Localization({
  en: enJson,
  ar: require('./ar.json'),
  be: require('./be@tarask.json'),
  // ... ~50 more eager require() calls
  zh_tw: require('./zh_tw.json'),
});

// After
const languageLoaders: Record<Exclude<LangCode, 'en'>, () => LanguageDict> = {
  ar: () => require('./ar.json'),
  be: () => require('./be@tarask.json'),
  // ...
};

export const parsedLanguages: Record<string, LanguageDict> = { en: enJson };
const loc = new Localization<typeof enJson>({ en: enJson }) as ILocalization;

const applyLanguage = (lang: string) => {
  if (lang === 'en' || !isKnownLang(lang)) {
    loc.setContent({ en: enJson });
    loc.setLanguage('en');
    return;
  }
  if (!parsedLanguages[lang]) {
    parsedLanguages[lang] = languageLoaders[lang]();
  }
  loc.setContent({ en: enJson, [lang]: parsedLanguages[lang] });
  loc.setLanguage(lang);
};
```

## Takeaway
Keep only the active locale (plus a fallback) in memory and load others through cached `require()` thunks. The PR notes this saves heap only, not bundle size or startup time.
