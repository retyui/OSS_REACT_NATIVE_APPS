# perf(i18n): load only active and fallback locales
- **Repository:** mrzmyr/pixy-mood-tracker-app
- **PR Link:** <https://github.com/mrzmyr/pixy-mood-tracker-app/pull/786>
- **Merged Date:** 2026-10-18
- **Category:** Startup

## Problem
At every launch the app evaluated 31 translation JSON files and 33 dayjs locales, and i18n-js deep-merged all 31 tables (about 20,500 strings), although only the device language and English are used. Counting evaluated locale modules in Jest, the PR reports 31 JSONs plus 33 dayjs locales before; after, a `de-DE` device loads 2 JSONs (`en`, `de`) and 1 dayjs locale, and an English device loads 1 JSON and 0 dayjs locales.

## Solution
- Replaced the eager `require` maps with `Map`s of loader functions. Metro module factories are lazy, so a `require` inside a loader only evaluates the module when the loader is called.
- `createI18n` stores only `en` (fallback) and the device language; dayjs locales are loaded the same way for the active language.
- `applyLocale` loads a new language on demand before switching, for the Android 13+ per-app language change.
- Behavior is preserved: language-part resolution (`pt-BR` to `pt`, `nb`/`nn` to `no.json`), English fallback for unsupported languages and missing keys. Tests cover these cases.

## Code Pattern (Before vs. After)
```ts
// Before
const translations = {
  ar: require("../../assets/locales/ar.json"),
  de: require("../../assets/locales/de.json"),
  // ... 31 tables evaluated at startup
};

const createI18n = (locale: string) => {
  const instance = new I18n(translations);
  // ...
};

// After
const translationLoaders = new Map<string, () => TranslationTable>([
  ["ar", () => require("../../assets/locales/ar.json")],
  ["de", () => require("../../assets/locales/de.json")],
  // ...
]);

const storeTranslations = (instance: I18n, locale: string) => {
  const language = toLanguage(locale);
  const load = translationLoaders.get(language);
  if (!load || storedLanguages.has(language)) {
    return;
  }
  instance.store({ [language]: load() });
  storedLanguages.add(language);
};

const createI18n = (locale: string) => {
  const instance = new I18n();
  storeTranslations(instance, "en");
  storeTranslations(instance, locale);
  // ...
};
```

## Takeaway
Wrap large per-locale resources in loader functions and load only the fallback and active locales, so startup does not evaluate or merge every translation.
