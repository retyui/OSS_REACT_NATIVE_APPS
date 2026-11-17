# perf: cache Intl instances
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/16151>
- **Merged Date:** 2025-06-05
- **Category:** Rendering

## Problem
Creating `new Intl.NumberFormat(...)` is expensive on Hermes, and every `TokenListItem` created several while formatting balances and fiat values. The PR (an internal version of a Margelo PR) reports `TokenList` render averaging about 250 ms before and about 170 ms after (30% faster, measured on a high-end device), and expects fewer blank cells when scrolling fast.

## Solution
- Add `getIntlNumberFormatter(locale, options)`, a drop-in replacement for `new Intl.NumberFormat(locale, options)` backed by a `Map` cache keyed by locale plus stringified options.
- Replace all `new Intl.NumberFormat` call sites (`formatWithThreshold`, `formatCurrency`, simulation-details formatters, ramp `formatAmount`).
- Unit tests check that identical arguments return the same instance and different ones do not.

## Code Pattern (Before vs. After)
```ts
// Before
return amount < threshold
  ? `<${new Intl.NumberFormat(locale, options).format(threshold)}`
  : new Intl.NumberFormat(locale, options).format(amount);

// After
return amount < threshold
  ? `<${getIntlNumberFormatter(locale, options).format(threshold)}`
  : getIntlNumberFormatter(locale, options).format(amount);
```

```ts
// app/util/intl.ts
const numberFormatCache = new Map<string, Intl.NumberFormat>();

export function getIntlNumberFormatter(
  locale: string,
  options?: Intl.NumberFormatOptions,
): Intl.NumberFormat {
  const cacheKey = `${locale}${options ? JSON.stringify(options) : ''}`;
  let formatter = numberFormatCache.get(cacheKey);
  if (!formatter) {
    formatter = Intl.NumberFormat(locale, options);
    numberFormatCache.set(cacheKey, formatter);
  }
  return formatter;
}
```

## Takeaway
Cache `Intl.NumberFormat` / `DateTimeFormat` instances per locale and options on Hermes instead of constructing them in render or per-row code.
