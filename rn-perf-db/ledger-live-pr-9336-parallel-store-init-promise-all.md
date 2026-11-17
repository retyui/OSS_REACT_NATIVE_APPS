# :thread: (llm): optimize launch
- **Repository:** LedgerHQ/ledger-live
- **PR Link:** <https://github.com/LedgerHQ/ledger-live/pull/9336>
- **Merged Date:** 2025-02-26
- **Category:** Startup

## Problem
`LedgerStoreProvider.init()` read persisted data (BLE, settings, accounts, market, trustchain, wallet, countervalues, ...) with a chain of sequential `await` calls, so each storage read had to finish before the next one started. This blocked the first render of the mobile app. The PR reports roughly a 30% reduction of the store provider init time, measured by timing the start and end of `init` (before/after videos, no per-call numbers).

## Solution
- All independent reads are started at once and awaited together with `Promise.all`.
- Each read is wrapped in a small `retry` helper (3 retries, 500 ms delay) so a transient storage failure does not break startup.
- Dependent work (hydrating currencies, computing supported countervalues, dispatching to the store) runs after the parallel reads resolve, so the app still starts only once everything is loaded.
- The class component was converted to a function component with `useEffect` + `useCallback`.
- The PR notes the gain depends on account count and device conditions; risk is low because only execution order of independent reads changes.

## Code Pattern (Before vs. After)
```tsx
// Before
async init() {
  const bleData = await getBle();
  this.props.store.dispatch(importBle(bleData));
  const settingsData = await getSettings();
  const cachedCurrencyIds = await listCachedCurrencyIds();
  // ...
  const accountsData = await getAccounts();
  const postOnboardingState = await getPostOnboardingState();
  const marketState = await getMarketState();
  const trustchainStore = await getTrustchainState();
  const walletStore = await getWalletExportState();
  const protect = await getProtect();
  const initialCountervalues = await getCountervalues();
}

// After
const [
  bleData, settingsData, cachedCurrencyIds, supportedFiats, accountsData,
  postOnboardingState, marketState, trustchainStore, walletStore, protect,
  initialCountervalues,
] = await Promise.all([
  retry(getBle, MAX_RETRIES, RETRY_DELAY),
  retry(getSettings, MAX_RETRIES, RETRY_DELAY),
  retry(listCachedCurrencyIds, MAX_RETRIES, RETRY_DELAY),
  retry(listSupportedFiats, MAX_RETRIES, RETRY_DELAY),
  retry(getAccounts, MAX_RETRIES, RETRY_DELAY),
  // ... remaining reads
  retry(getCountervalues, MAX_RETRIES, RETRY_DELAY),
]);
```

## Takeaway
Audit startup code for chains of independent `await`s (storage, MMKV/AsyncStorage reads, config loads) and run them with `Promise.all`.
