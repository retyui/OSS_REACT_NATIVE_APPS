# Dynamically load `date-fns` locale data
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/9535>
- **Merged Date:** 2026-02-18
- **Category:** Bundle Size (+ Startup)

## Problem
`dates.ts` statically imported ~35 `date-fns` locale objects (`ca`, `cy`, `de`, ... `zhTW`) from `date-fns/locale`, so every user parsed and shipped all of them even though only one language is active. The PR states that making this dynamic "should save us almost 400kb" (before/after bundle screenshots are attached to the PR).

## Solution
- Removed the static locale map from `useFormatDistance`.
- `dynamicActivate(locale)` in `i18n.ts`, which already lazy-loads the formatjs locale data per language, now also does `import('date-fns/locale/<x>')` in the same `Promise.all` and returns the date-fns locale.
- `useLocaleLanguage()` returns that locale, and `I18nProvider` exposes it through a `DateLocaleContext` with a `useDateLocale()` hook.
- `useFormatDistance` reads the locale from context, so callers are unchanged. English uses a statically imported `en-US` default.
- Native and web i18n entry points were both updated.

## Code Pattern (Before vs. After)
```tsx
// Before (src/components/hooks/dates.ts)
import {formatDistance, type Locale} from 'date-fns'
import {ca, cy, da, de, /* ... */ zhTW} from 'date-fns/locale'

const locales: Record<AppLanguage, Locale | undefined> = {
  en: undefined,
  ca,
  cy,
  // ...
}

export function useFormatDistance() {
  const {appLanguage} = useLanguagePrefs()
  return React.useCallback<typeof formatDistance>(
    (date, baseDate, options) => {
      const locale = locales[appLanguage as AppLanguage]
      return formatDistance(date, baseDate, {...options, locale: locale})
    },
    [appLanguage],
  )
}

// After
// src/locale/i18n.ts
case AppLanguage.de: {
  i18n.loadAndActivate({locale, messages: messagesDe})
  const [{default: dateLocale}] = await Promise.all([
    import('date-fns/locale/de'),
    import('@formatjs/intl-pluralrules/locale-data/de'),
    import('@formatjs/intl-numberformat/locale-data/de'),
    import('@formatjs/intl-displaynames/locale-data/de'),
  ])
  return dateLocale
}

// src/components/hooks/dates.ts
export function useFormatDistance() {
  const locale = useDateLocale()
  return useCallback<typeof formatDistance>(
    (date, baseDate, options) => formatDistance(date, baseDate, {...options, locale}),
    [locale],
  )
}
```

## Takeaway
Any per-language data table (date, number, plural rules) should be loaded for the active language only, alongside the existing i18n bootstrap, and delivered via context.
