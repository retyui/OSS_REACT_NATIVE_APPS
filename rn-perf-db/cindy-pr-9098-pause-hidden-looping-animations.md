# perf(mobile): 暂停不可见页面的循环动画
- **Repository:** makecindy/cindy
- **PR Link:** <https://github.com/makecindy/cindy/pull/9098>
- **Merged Date:** 2026-11-06
- **Category:** Animations

## Problem
The retained (kept-mounted) home list kept running spinner and breathing animations after the user navigated to another screen, which kept triggering Android view traversals and draws. In an Android API 36 emulator (dev build, Hermes, mock data, 600-row list at rest, last 8 s of a 12 s trace), traversal and draw counts went from 479 to 0, main-thread CPU from 932.63 ms to about 414-424 ms, and RenderThread CPU from 914.65 ms to 0 ms. The PR notes that framework animation callbacks did not disappear and that emulator results do not translate to device battery or release frame rates.

## Solution
- Added a `useLoopingAnimation()` hook that returns true only when the surface is visible (via an `AnimationVisibilityProvider` context), the app is in the foreground (a single shared `AppState` listener via `useSyncExternalStore`), and reduced motion is explicitly off.
- Looping components (task spinner, vendor icon breathing, connection dot, list status breathing, sync indicator, companion presence ring) now start `Animated.loop` only when this hook is true and stop it otherwise, keeping their state and content.
- The retained home list wraps itself in the provider with its visibility, so a hidden list stops animating but keeps its scroll position; a visible wide-screen sidebar keeps animating.
- iOS native-stack headers render outside the route's providers, so a bridge hook captures the visibility and carries it into the header content.
- iOS behavior was verified only by tests and review, not on a simulator.

## Code Pattern (Before vs. After)
```tsx
// Before
const reduceMotion = useReduceMotionEnabled();
useEffect(() => {
  if (reduceMotion !== false) return;
  const loop = Animated.loop(/* spin timing */);
  // ...
  return () => { loop.stop(); spin.setValue(0); };
}, [reduceMotion, spin]);

// After
export function useLoopingAnimation(enabled = true): boolean {
  const visible = useContext(AnimationVisibility);
  const foreground = useSyncExternalStore(subscribeForeground, isForeground, serverForeground);
  const reduceMotion = useReduceMotionEnabled();
  return enabled && visible && foreground && reduceMotion === false;
}

// in the component
const animate = useLoopingAnimation();
useEffect(() => {
  if (!animate) return;
  const loop = Animated.loop(/* spin timing */);
  // ...
  return () => { loop.stop(); spin.setValue(0); };
}, [animate, spin]);
```

## Takeaway
Gate every infinite animation on screen visibility and app foreground state; screens kept mounted for fast return still cost draw passes if their loops keep running.
