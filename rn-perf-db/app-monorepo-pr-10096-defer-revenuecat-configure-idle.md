# fix: defer RevenueCat configure to avoid iOS AppHang during startup
- **Repository:** OneKeyHQ/app-monorepo
- **PR Link:** <https://github.com/OneKeyHQ/app-monorepo/pull/10096>
- **Merged Date:** 2026-02-10
- **Category:** Startup

## Problem
`PurchasesReactNative.configure()` runs RevenueCat's native `setupPurchases` synchronously on the main thread via a TurboModule. It decodes the cached `CustomerInfo` JSON (including URL parsing), which the PR reports caused an AppHang of 5s or more on lower-memory iOS devices (for example an iPhone XR with 33MB free memory).

## Solution
- Wrapped the `configure()` call and the `setIsPaymentReady(true)` update in `requestIdleCallback`, so they run after the first frame and startup work.
- Prime features are not needed on the first frame, so the PR notes the delay has no functional impact.
- Payment readiness is only flagged once configuration has actually happened.

## Code Pattern (Before vs. After)
```ts
// Before
const { apiKey } = await getPrimePaymentApiKey({ apiKeyType: 'native' });
PurchasesReactNative.configure({
  apiKey,
  // useAmazon: true
});
setIsPaymentReady(true);

// After
const { apiKey } = await getPrimePaymentApiKey({ apiKeyType: 'native' });
// Defer RevenueCat configure to avoid blocking main thread during startup.
requestIdleCallback(() => {
  PurchasesReactNative.configure({
    apiKey,
    // useAmazon: true
  });
  setIsPaymentReady(true);
});
```

## Takeaway
Defer third-party SDK initialization that runs synchronously on the main thread until the app is idle, and gate dependent features on a readiness flag.
