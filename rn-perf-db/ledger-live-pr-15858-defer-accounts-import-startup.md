# [LWM] perf(lwm): shorten startup by deferring the accounts import
- **Repository:** LedgerHQ/ledger-live
- **PR Link:** <https://github.com/LedgerHQ/ledger-live/pull/15858>
- **Merged Date:** 2026-04-03
- **Category:** Startup

## Problem
Since the crypto-asset-list (CAL) removal, store initialisation contained several blocking network calls (token cache restore and accounts import) that ran before the app could render. Their cost scales with the user's number of accounts and network quality, so the impact is largest on slow networks and with many accounts. The PR shows before/after videos but reports no numbers.

## Solution
- `LedgerStoreProvider` no longer imports accounts during init. It builds an `importAccounts` function (token hydration, then `importAccountsRaw`) and passes it down through the children render-prop.
- `WaitForAppReady` runs `importAccounts()` and treats it as one more readiness condition alongside currency init, OFAC and Firebase. The existing `MAX_WAIT` of 1 s (timeout) ensures a slow network cannot hold the splash screen indefinitely.
- Errors during import are caught and logged so deeplink navigation is not blocked.
- A TODO notes that token cache restoration might also be made non-blocking.

## Code Pattern (Before vs. After)
```tsx
// Before (inside LedgerStoreProvider init)
if (cryptoAssetsCache?.tokens) {
  await restoreTokensToCache(store.dispatch, cryptoAssetsCache, TOKEN_CACHE_TTL);
}
try {
  store.dispatch(await importAccountsRaw(accountsData));
} catch (error) {
  console.error("Failed to import accounts during initialization:", error);
}

// After
importAccounts.current = getImportAccounts(
  store,
  hydrateCryptoAssets(store, cryptoAssetsCache),
  accountsData,
);
```

```tsx
// WaitForAppReady.tsx
const accountsImported = useWait<boolean>(resolve =>
  importAccounts().finally(() => resolve(true)),
);
const isLoaded =
  accountsImported &&
  currencyInitialized &&
  !initialQueries.ofacResult.isLoading &&
  initialQueries.firebaseIsReady;
const timedOut = useWait<boolean>(resolve => setTimeout(() => resolve(true), MAX_WAIT)) ?? false;
```

## Takeaway
Move data hydration that depends on network or account count out of the store-init critical path, and gate the splash screen on it with a hard timeout.
