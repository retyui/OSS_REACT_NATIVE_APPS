# perf: lazy tab loading, idle-time preload & device-tier-aware startup for low-end devices
- **Repository:** OneKeyHQ/app-monorepo
- **PR Link:** <https://github.com/OneKeyHQ/app-monorepo/pull/10819>
- **Merged Date:** 2026-07-01
- **Category:** Startup (+ Rendering)

## Problem
With `lazy: false` on the native tab navigator, the first paint mounted every tab screen, which hurt cold start and tab navigation, especially on low-memory devices. On low-memory iOS, jetsam also killed the app in a cold-start-lock loop. The PR reports no measured timings; it describes the work qualitatively.

## Solution
- Native tab screens now use `lazy: true`, so only the focused tab mounts on first paint.
- Key tabs are then preloaded in the background, one per step, spaced by an interval and run inside `requestIdleCallback`, driven by a per-platform, per-tier queue in `preloadConfig.ts`. The `low` tier does no preloading.
- A new device performance tier (`high`, `medium`, `low`) is read synchronously from a cached value or device memory, and is re-calibrated asynchronously 5s after the UI is visible for the next launch.
- A single `LOW_END_MEMORY_THRESHOLD_GB = 3.5` is shared by the tier, the DApp WebView alive-count cap and the cold-start-lock defer, so these decisions agree.
- On low-memory iOS, the app tree render is deferred behind the cold-start lock until first unlock.
- IndexedDB buckets now open in parallel with `Promise.all`; this part affects web, not React Native.
- Caveats from the code: preload dispatches must use `target` and no params, or they crash the StackView or remount screens. On native, the `requestIdleCallback` shim is a 1ms `setTimeout` with no real idle detection, so pacing comes from the interval timer.

## Code Pattern (Before vs. After)
```tsx
// Before (TabStackNavigator.native.tsx)
const nativeTabScreenOptions = {
  freezeOnBlur: !platformEnv.isNativeIOS,
  preventsDefault: false,
  lazy: false,
};

// After
const nativeTabScreenOptions = {
  freezeOnBlur: !platformEnv.isNativeIOS,
  preventsDefault: false,
  lazy: true,
};
```

```tsx
// After (Tab/Navigator.tsx): progressive idle-time preload
const tier = getDevicePerformanceTier();
const { queue: preloadQueue, intervalMs: PRELOAD_INTERVAL_MS } =
  tabPreloadConfig[tier] ?? defaultPreloadEntry;
if (preloadQueue.length === 0) return;

function scheduleNext() {
  timerId = setTimeout(() => {
    idleHandle = requestIdleCallback(preloadNext);
  }, PRELOAD_INTERVAL_MS);
}
// ...
rootNavigationRef.current?.dispatch({
  ...CommonActions.preload(preloadQueue[index]),
  target: tabStateKey,
});
```

```ts
// After (preloadConfig.ts)
const nativePreloadConfig: Record<string, IPreloadEntry> = {
  [EDevicePerformanceTier.high]: {
    queue: [ETabRoutes.Swap, ETabRoutes.Discovery, ETabRoutes.Perp],
    intervalMs: 2000,
  },
  [EDevicePerformanceTier.medium]: {
    queue: [ETabRoutes.Swap, ETabRoutes.Perp],
    intervalMs: 3000,
  },
};
```

## Takeaway
Mount only the first tab at launch, then warm the others gradually, and scale that warm-up down (to nothing) on low-RAM devices.
