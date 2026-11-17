# perf: lazy-load rare-trigger global modals to shrink ManualAppStartup
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/89180>
- **Merged Date:** 2026-05-08
- **Category:** Startup

## Problem
`UpdateAppModal`, `ScreenShareRequestModal` and `ProactiveAppReviewModalManager` were statically imported by `GlobalModals`, so their module parse, hook execution and `useOnyx` subscription registration all ran inside the `ManualAppStartup` span, although they only show in rare situations. The PR gives no before/after numbers.

## Solution
- Load the three modals with `React.lazy` and mount them only after the existing `requestIdleCallback` (with a 2 s timeout) that already defers the context menu, wrapped in `startTransition`. No new listener is added.
- New `LazyModalSlot` wraps each lazy child in its own `ErrorBoundary` plus `Suspense`, so a chunk-load failure in one slot cannot break siblings; failures are logged.
- JSX order is kept deliberate: `BaseModal` hardcodes `zIndex: 1`, so `UpdateAppModal` is last to stay on top.
- The context-menu "ensure mounted" registration moved to its own effect so eager mounting on user interaction still works.

## Code Pattern (Before vs. After)
```tsx
// Before
import ProactiveAppReviewModalManager from './components/ProactiveAppReviewModalManager';
import ScreenShareRequestModal from './components/ScreenShareRequestModal';
import UpdateAppModal from './components/UpdateAppModal';
// ...
<UpdateAppModal />
<ProactiveAppReviewModalManager />
<ScreenShareRequestModal />

// After
const LazyUpdateAppModal = React.lazy(() => import('./components/UpdateAppModal'));
const LazyScreenShareRequestModal = React.lazy(() => import('./components/ScreenShareRequestModal'));
const LazyProactiveAppReviewModalManager = React.lazy(() => import('./components/ProactiveAppReviewModalManager'));
// ...
const id = requestIdleCallback(() => {
    startTransition(() => {
        setShouldRenderContextMenu(true);
        setShouldRenderDeferredModals(true);
    });
}, {timeout: IDLE_CALLBACK_TIMEOUT_MS});
// ...
{shouldRenderDeferredModals && (
    <>
        <LazyModalSlot><LazyProactiveAppReviewModalManager /></LazyModalSlot>
        <LazyModalSlot><LazyScreenShareRequestModal /></LazyModalSlot>
        <LazyModalSlot><LazyUpdateAppModal /></LazyModalSlot>
    </>
)}
```

## Takeaway
Any always-mounted component that only matters in rare conditions can be lazy-loaded after first paint, ideally with its own error boundary and Suspense slot.
