# Defer startup imports
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/84390>
- **Merged Date:** 2026-03-11
- **Category:** Startup

## Problem
Four non-critical modules were imported statically at startup. Two of them (`UnreadIndicatorUpdater`, `replaceOptimisticReportWithActualReport`) register `Onyx.connectWithoutView` subscriptions with `waitForCollectionCallback: true` on large collections (reports, report actions, draft comments), which compete for storage I/O with the Onyx reads that gate the splash screen (session, public-room check, preferred locale). The PR reports no numbers.

## Solution
Remove the static imports and load `UnreadIndicatorUpdater`, `replaceOptimisticReportWithActualReport` and `registerPaginationConfig` with dynamic `import()` in a `useEffect` in `SplashScreenStateContext` once the state becomes `HIDDEN`. `subscribeToPushNotifications` is imported dynamically in `setup/index.ts` right after `Onyx.init()`, outside the React lifecycle, so push handlers are registered before any push arrives (including Android headless wake-ups). Deferring is safe because `waitForCollectionCallback` delivers a full snapshot on first subscribe, so no updates are missed.

## Code Pattern (Before vs. After)
```tsx
// Before: Expensify.tsx
import './libs/actions/replaceOptimisticReportWithActualReport';
import './libs/registerPaginationConfig';
import './libs/UnreadIndicatorUpdater';

// After: SplashScreenStateContext.tsx
function loadPostSplashScreenModules() {
    import('./libs/actions/replaceOptimisticReportWithActualReport');
    import('./libs/registerPaginationConfig');
    import('./libs/UnreadIndicatorUpdater');
}

useEffect(() => {
    if (splashScreenState !== CONST.BOOT_SPLASH_STATE.HIDDEN) {
        return;
    }
    loadPostSplashScreenModules();
}, [splashScreenState]);
```
