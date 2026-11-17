# Hide/show header and footer without re-renders, take two
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/1849>
- **Merged Date:** 2023-11-09
- **Category:** Rendering (+ Animations)

## Problem
Hiding and showing the header, footer and FAB on scroll was stored as React state (`minimalShellMode`), so every toggle re-rendered all consumers (top bar, bottom bar, thread screen, scroll handler). An earlier fix had regressed. The impact was on low/mid-range Android; the PR shows before/after videos and a render-counting patch to React Native's renderer, but no numbers in text.

## Solution
- Make a Reanimated `SharedValue<boolean>` the source of truth in the shell-mode context instead of `useState`.
- `setMinimalShellMode` writes `mode.value = v`, which triggers no React render.
- Animated styles read `mode.value` and apply `withTiming` inside `useAnimatedStyle`, replacing the `autorun` + effect + extra interpolation shared value.
- `pointerEvents` moved into the animated style, so the `disabled` styles and the `minimalShellMode` reads in render were removed.
- The PostThread reply prompt is now always mounted and animated with the FAB transform instead of conditionally rendered.
- Value is still updated discretely, not continuously with scroll.

## Code Pattern (Before vs. After)
```tsx
// Before
const [state, setState] = React.useState(false)
// ...
React.useEffect(() => {
  return autorun(() => {
    minimalShellInterp.value = withTiming(minimalShellMode ? 1 : 0, {duration: 125, /* ... */})
  })
}, [minimalShellInterp, minimalShellMode])
// ...
{isMobile && !minimalShellMode && <View style={styles.prompt}>...</View>}

// After
const mode = useSharedValue(false)
const setMode = React.useCallback((v: boolean) => { mode.value = v }, [mode])
// ...
const footerMinimalShellTransform = useAnimatedStyle(() => ({
  pointerEvents: mode.value ? 'none' : 'auto',
  opacity: withShellTiming(interpolate(mode.value ? 1 : 0, [0, 1], [1, 0])),
  transform: [{translateY: withShellTiming(interpolate(mode.value ? 1 : 0, [0, 1], [0, 25]))}],
}))
// ...
{isMobile && (
  <Animated.View style={[styles.prompt, fabMinimalShellTransform]}>...</Animated.View>
)}
```

## Takeaway
Keep UI-only toggles that only drive animations in shared values so changing them does not re-render the React tree.
