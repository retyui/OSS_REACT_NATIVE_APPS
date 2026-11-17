# Freeze non top screens to prevent extra-rerenders 2
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/85443>
- **Merged Date:** 2026-04-07
- **Category:** Rendering

## Problem
Screens that are covered by another screen in the navigation stack keep re-rendering whenever shared state (Onyx, contexts) changes, even though the user cannot see them. This is wasted work, notably for the split navigator's sidebar and chat screens. The PR re-applies an earlier freeze change (#82764) that was reverted (#85198) because of several regressions. It reports no timing numbers.

## Solution
- Added a `freezeNonTopScreens` option to `createPlatformStackNavigatorComponent`, enabled for the split navigator.
- `wrapDescriptorsWithFreeze` wraps every screen's `render` in a `ScreenFreezeWrapper` that uses `react-freeze` (`<Freeze freeze={...}>`) when the screen is not on top. Persistent screens (such as the web sidebar) are excluded.
- Fixes for the earlier regressions: web freezing is deferred by one frame if a modal or tooltip is open, or if any component registered a "freeze defer" via the new `ScreenFreezeContext`. `useKeyboardShortcut` registers one so its unsubscribe cleanup runs before the screen freezes. Several components that subscribed to shortcuts by hand were moved onto that hook.
- On native, freezing is delayed by 500 ms so transitions and popover dismiss animations can finish, and the content is wrapped in a `display: contents` view so the underlay stays visible during swipe-back.
- Risk noted in the diff: the native wrapper uses internal RN APIs (`NativeComponentRegistry`, `ReactNativeStyleAttributes`) validated on RN 0.83.1.

## Code Pattern (Before vs. After)
```tsx
// Before
<NavigationContent>
    <Navigator state={state} descriptors={descriptors} navigation={navigation} describe={describe} />
</NavigationContent>

// After
const wrappedDescriptors = freezeNonTopScreens ? wrapDescriptorsWithFreeze(descriptors, state) : descriptors;
// ...
<Navigator state={state} descriptors={wrappedDescriptors} navigation={navigation} describe={describe} />

// wrapDescriptorsWithFreeze
const topRouteKey = state.routes[state.index]?.key;
for (const [key, descriptor] of Object.entries(descriptors)) {
    const isOnTop = key === topRouteKey;
    const isPersistent = persistentScreens?.includes(descriptor.route.name);
    const isScreenBlurred = !isOnTop && !isPersistent;
    result[key] = {
        ...descriptor,
        render: () => <ScreenFreezeWrapper isScreenBlurred={isScreenBlurred}>{descriptor.render()}</ScreenFreezeWrapper>,
    };
}
```

```tsx
// ScreenFreezeWrapper (native): freeze only after transitions settle
const id = setTimeout(() => {
    rafID = requestAnimationFrame(() => setFrozen(true));
}, FREEZE_DELAY_MS);
// ...
<Freeze freeze={frozen}>
    <CustomViewWrapper style={styles.flex1}>{children}</CustomViewWrapper>
</Freeze>
```

## Takeaway
Freezing covered screens with `react-freeze` removes background re-renders, but delay the freeze until animations and cleanups finish, and unfreeze synchronously.
