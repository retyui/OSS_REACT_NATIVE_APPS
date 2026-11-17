# Freeze non top screens to prevent extra-rerenders
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/82764>
- **Merged Date:** 2026-03-12
- **Category:** Rendering

## Problem
Screens that are in the navigation stack but not on top (hidden under a report, modal or settings page) kept re-rendering as store data changed, wasting work on UI the user cannot see. The PR gives no measurements.

## Solution
- Web: a new `ScreenFreezeWrapper` wraps non-top screens of the split navigator (enabled via `freezeNonTopScreens: true`) in `react-freeze`'s `Freeze`. The freeze is applied in a layout effect and, when a tooltip is active, deferred one frame so the tooltip can dismiss first.
- Native: a `@react-navigation/native-stack` patch removes the Fabric exception that kept the screen below the focused one unfrozen, and a `react-native-screens` patch makes `DelayedFreeze` wait for `requestAnimationFrame` so a frame paints before freezing, avoiding flicker.
- Caveat: this change shipped, then was reverted (Expensify/App#85198) over regressions, and the PR #85443 re-landed it with fixes. Treat freezing as a pattern that needs careful testing of modals, tooltips and underlying-screen visibility.

## Code Pattern (Before vs. After)
```js
// Before (native-stack patch target)
// On Fabric, when screen is frozen, animated and reanimated values are not updated
// due to component being unmounted. To avoid this, we don't freeze the previous screen there
const shouldFreeze = isFabric() ? !isPreloaded && !isFocused && !isBelowFocused : !isPreloaded && !isFocused;

// After
// Freezing the screen below the focused one is safe on Fabric because
// DelayedFreeze defers it to the next macrotask, and transition animations run on the UI thread.
const shouldFreeze = !isPreloaded && !isFocused;
```

```tsx
// After: ScreenFreezeWrapper (web)
function ScreenFreezeWrapper({isScreenBlurred, children}: ScreenFreezeWrapperProps) {
    const [frozen, setFrozen] = useState(false);

    useLayoutEffect(() => {
        if (!TooltipSense.isActive() || !isScreenBlurred) {
            setFrozen(isScreenBlurred);
        }
        const id = requestAnimationFrame(() => setFrozen(isScreenBlurred));
        return () => cancelAnimationFrame(id);
    }, [isScreenBlurred]);

    return <Freeze freeze={frozen}>{children}</Freeze>;
}
```
