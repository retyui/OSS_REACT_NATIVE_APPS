# Avoid per-frame Reanimated commits in the profile header
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/12161>
- **Merged Date:** 2026-10-23
- **Category:** Animations (+ Native)

## Problem
Scrolling a profile committed the React Native shadow tree on almost every frame, even after the header stopped moving. Reanimated re-sends an animated style unless every key is `===` to the previous one: `PagerTabBar` returned a new `transform` array each run, and `MinimalHeader` returned `withTiming(...)` for opacity every run (which skips the comparison and restarts the 75 ms fade each frame). On a Galaxy A16 (release, Perfetto, 8 interleaved runs per arm) Reanimated commits per scrolling frame dropped from 0.98 to 0.08 (drag) and 0.88 to 0.03 (fling); UI-thread CPU fell 6.99 s to 5.94 s (-15%) and 6.27 s to 4.62 s (-26%). Dropped frames were already near zero, so the gain is UI-thread headroom.

## Solution
- Compute offsets with `useDerivedValue`, which only notifies the style when the value actually changes, and read the derived value in `useAnimatedStyle`.
- In `MinimalHeader`, move `withTiming` out of the style into a `useAnimatedReaction` that sets a shared `opacity` only when crossing the scroll threshold.
- Apply the same to `GrowableBanner`, `GrowableAvatar` and `StatusBarShadow`, which animate on iOS.
- Alternatives (fixing Reanimated's comparison, caching the transform array) were rejected as larger or more fragile.

## Code Pattern (Before vs. After)
```tsx
// Before (MinimalHeader)
const animatedStyle = useAnimatedStyle(() => {
  const pastThreshold = scrollY.get() > 100
  return {
    opacity: pastThreshold ? withTiming(1, {duration: 75}) : withTiming(0, {duration: 75}),
    transform: [
      {translateY: Math.min(scrollY.get(), headerHeight - minimalHeaderHeight)},
    ],
  }
})

// After
const opacity = useSharedValue(0)
const translateY = useDerivedValue(() =>
  Math.min(scrollY.get(), headerHeight - minimalHeaderHeight),
)
const animatedStyle = useAnimatedStyle(() => ({
  opacity: opacity.get(),
  transform: [{translateY: translateY.get()}],
}))
useAnimatedReaction(
  () => scrollY.get() > 100,
  (value, prev) => {
    if (prev !== value) {
      opacity.set(withTiming(value ? 1 : 0, {duration: 75}))
      scheduleOnRN(setVisible, value)
    }
  },
)
```

## Takeaway
Do not return new `transform` arrays or `withTiming` from a `useAnimatedStyle` that reads scroll offset; derive the scalar first and trigger animations from a reaction.
