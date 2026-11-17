# refactor(InfiniteDiscovery): reduce usage of animatedStyle hooks for performance sake
- **Repository:** artsy/eigen
- **PR Link:** <https://github.com/artsy/eigen/pull/11652>
- **Merged Date:** 2025-03-11
- **Category:** Animations

## Problem
Each card in the Infinite Discovery swiper registered five separate `useAnimatedStyle` hooks (top, second, unswiped, last-swiped, swiped), plus a `useSharedValue` mirror of `index` synced through `useEffect`. Every card ran all five style worklets on every gesture frame, even though only one applied at a time. The PR gives no measurements; it states the goal as better performance and readability.

## Solution
- Compute small `useDerivedValue` flags (`isTopCard`, `isSecondCard`, `isSwiped`, `isThirdOrMoreCard`, `isLastSwiped`) from `activeIndex` and `swipedKeys`.
- Derive each animated property (translateX/Y, rotation, scale, opacity, shadow, elevation) from those flags with a single `useDerivedValue` each.
- Combine everything into one `useAnimatedStyle`, which also removes the per-card `emptyShadowStyle` workaround and the `_index` shared value and effect.
- Note: the rotation `useDerivedValue` as merged has no `return`, so rotation is not actually applied; that is a bug in the PR as merged and not part of the optimization.

## Code Pattern (Before vs. After)
```tsx
// Before
const _index = useSharedValue(index)
useEffect(() => {
  if (index !== _index.value) _index.value = index
}, [index])

const topCardStyle = useAnimatedStyle(() => { /* ... */ })
const secondCardStyle = useAnimatedStyle(() => { /* ... */ })
const unswipedCardStyle = useAnimatedStyle(() => { /* ... */ })
const lastSwipedCardStyle = useAnimatedStyle(() => { /* ... */ })
const swipedCardStyle = useAnimatedStyle(() => { /* ... */ })

<Animated.View style={[topCardStyle, secondCardStyle, unswipedCardStyle, lastSwipedCardStyle, swipedCardStyle, { width: screenWidth }]} />

// After
const isTopCard = useDerivedValue(() => activeIndex.value === index)
const isSecondCard = useDerivedValue(() => activeIndex.value - 1 === index)
const isSwiped = useDerivedValue(() => swipedKeys.value.includes(card.key as Key))
const isLastSwiped = useDerivedValue(() => activeIndex.value + 1 === index)

const transformX = useDerivedValue(() => {
  if (isTopCard.value) return Math.min(0, activeCardX.value)
  if (isLastSwiped.value) return swipedCardX.value
  if (isSwiped.value) return -width
  return 0
})
// ...scale, opacity, shadowOpacity, elevation likewise

const animatedStyle = useAnimatedStyle(() => ({
  transform: [
    { translateX: transformX.value },
    { translateY: transformY.value },
    { rotate: `${rotation.value}deg` },
    { scale: scale.value },
  ],
  shadowOpacity: shadowOpacity.value,
  elevation: elevation.value,
  opacity: opacity.value,
}))

<Animated.View style={[animatedStyle, { width: screenWidth }]} />
```

## Takeaway
For many instances of an animated item, prefer one `useAnimatedStyle` fed by cheap `useDerivedValue` state flags over several conditional style hooks that all run every frame.
