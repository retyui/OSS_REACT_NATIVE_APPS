# Cache Intl instances and replace usages to toLocaleString
- **Repository:** rainbow-me/rainbow
- **PR Link:** <https://github.com/rainbow-me/rainbow/pull/6773>
- **Merged Date:** 2025-07-29
- **Category:** Startup (+ Rendering)

## Problem
A profile of app launch plus opening the swap screen showed a large share of time in `toLocaleString`, which creates a new `Intl` formatter on each call. On Hermes for Android, creating formatters is very slow. The PR says the change saves around 400 ms.

## Solution
- New `src/helpers/intl.ts` with `getDateFormatter` and `getNumberFormatter`, which cache `Intl.DateTimeFormat` / `Intl.NumberFormat` instances by locale plus sorted-options key.
- The cache lives on `globalThis` and the helpers are worklets, because worklets cannot capture closure variables.
- `toLocaleString(...)` calls in number, date and points/rewards formatting code are replaced with `getNumberFormatter(...).format(...)` / `getDateFormatter(...)`.

## Code Pattern (Before vs. After)
```ts
// Before
return Number(value).toLocaleString('en-US', {
  useGrouping: true,
  minimumFractionDigits: Math.min(2, dec),
  maximumFractionDigits: dec,
});

// After
return getNumberFormatter('en-US', {
  maximumFractionDigits: dec,
  minimumFractionDigits: Math.min(2, dec),
  useGrouping: true,
}).format(Number(value));

// intl.ts
export function getNumberFormatter(locale?: string, options?: Intl.NumberFormatOptions): Intl.NumberFormat {
  'worklet';
  globalThis._numberFormatterCache ??= {};
  const key = getStableKey(locale, options);
  let formatter = globalThis._numberFormatterCache[key];
  if (!formatter) {
    formatter = new Intl.NumberFormat(locale, options);
    globalThis._numberFormatterCache[key] = formatter;
  }
  return formatter;
}
```

## Takeaway
Avoid `toLocaleString` in hot paths on Hermes; construct `Intl` formatters once and reuse them.
