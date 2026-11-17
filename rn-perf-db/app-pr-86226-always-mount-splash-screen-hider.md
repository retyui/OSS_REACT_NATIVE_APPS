# performance: Always mount `SplashScreenHider` and control hide timing via prop instead of conditional rendering
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/86226>
- **Merged Date:** 2026-03-26
- **Category:** Startup

## Problem
`SplashScreenHider` was only mounted once `shouldHideSplash` became true, and on native it then waited for the logo SVG `onLoadEnd` before starting the hide animation. Every cold start paid mount time plus SVG load time before the splash could begin to dismiss. The author reports lowering the `ManualAppStartup` metric by about 250 ms (from an initial ~3 s) in a release build on a Samsung Galaxy Z Flip 5.

## Solution
- Mount `SplashScreenHider` from the start and pass a `shouldHideSplash` prop; a `useEffect` fires the hide when it flips to true.
- Remove the native `onLoadEnd` trigger, since the logo is already loaded by then.
- Unmount the component after `onHide` completes (tracked with a `splashHideHasBeenCalled` state in `Expensify.tsx`).
- Use `useEffectEvent` for the hide callback to avoid stale closures without listing it as a dependency.

## Code Pattern (Before vs. After)
```tsx
// Before
{shouldHideSplash && <SplashScreenHider onHide={onSplashHide} />}

// SplashScreenHider (native)
<ImageSVG onLoadEnd={hide} contentFit="fill" src={Logo} />

// After
{!splashHideHasBeenCalled && (
    <SplashScreenHider
        shouldHideSplash={shouldHideSplash}
        onHide={onSplashHide}
    />
)}

// SplashScreenHider
const hide = useEffectEvent(() => { /* start hide animation */ });
useEffect(() => {
    if (!shouldHideSplash) {
        return;
    }
    hide();
}, [shouldHideSplash]);
```

## Takeaway
Keep latency-critical overlays mounted and assets preloaded, and drive them with a prop so the hide starts the moment the app is ready.
