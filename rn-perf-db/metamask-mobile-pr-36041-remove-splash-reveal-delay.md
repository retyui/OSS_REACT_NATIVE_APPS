# perf: drop the 250ms frozen frame from the splash reveal
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/36041>
- **Merged Date:** 2026-09-21
- **Category:** Startup (+ Animations)

## Problem
`ControllersGate` waited 250 ms after the splash animation finished before starting the fade to the app. A frame-by-frame cold-start capture showed that window was a completely frozen, blank screen: 20 consecutive recorded frames with zero pixel difference, spanning 225 ms, while the app was already rendered underneath. The PR reports the longest pixel-identical run across the reveal dropping from 225 ms (20 frames) to 22 ms (2 frames).

## Solution
The fade now starts immediately once both the animation is done and app services are ready. The 300 ms fade and the 800 ms Rive exit constant were measured and deliberately left alone: removing them would cut off real visual motion. A unit test guards against re-introducing a delay before the fade.

## Code Pattern (Before vs. After)
```tsx
// Before
useEffect(() => {
  if (animationDone && appServicesReady) {
    const timer = setTimeout(fadeOutLoader, 250);
    return () => clearTimeout(timer);
  }
  return undefined;
}, [animationDone, appServicesReady, fadeOutLoader]);

// After
useEffect(() => {
  if (animationDone && appServicesReady) {
    fadeOutLoader();
  }
}, [animationDone, appServicesReady, fadeOutLoader]);
```

## Takeaway
Measure startup timers against captured frames; fixed `setTimeout` delays on the cold path are often pure latency.
