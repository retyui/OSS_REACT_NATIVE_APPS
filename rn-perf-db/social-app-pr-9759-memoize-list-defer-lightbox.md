# [Lighbox perf - 1] Fix memoization of List component, defer re-rendering when opening lightbox
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/9759>
- **Merged Date:** 2026-01-28
- **Category:** Rendering (+ Lists)

## Problem
`<List>` passed a plain wrapper (`handleScrolledDownChange`) into `useAnimatedScrollHandler`. React Compiler does not memoize it because Reanimated's Babel transform generates the dependency array itself (and strips a manual `useCallback`), so `scrollHandler` changed identity on every render and the FlatList re-rendered. This hurt while opening the image lightbox. The PR says it saves "10s of milliseconds" at that moment (before/after screenshots only, no table of numbers).

## Solution
- Wrap the handler in `useNonReactiveCallback` so its identity is guaranteed stable.
- Wrap `Animated.FlatList` in `memo` (`FlatList_INTERNAL`).
- Apply `useDeferredValue` to the lightbox-dependent `scrollsToTop` value, so when the FlatList must re-render for that prop it does so at lower priority after the animation work.

## Code Pattern (Before vs. After)
```tsx
// Before
function handleScrolledDownChange(didScrollDown: boolean) {
  onScrolledDownChange?.(didScrollDown)
}
// ...
export const FlatList_INTERNAL = Animated.FlatList
// ...
function useAllowScrollToTopIOS() {
  const {activeLightbox} = useLightbox()
  return !activeLightbox
}

// After
const handleScrolledDownChange = useNonReactiveCallback(
  (didScrollDown: boolean) => {
    onScrolledDownChange?.(didScrollDown)
  },
)
// ...
export const FlatList_INTERNAL = memo(Animated.FlatList)
// ...
function useAllowScrollToTopIOS() {
  const {activeLightbox} = useLightbox()
  return useDeferredValue(!activeLightbox)
}
```

## Takeaway
Callbacks passed into Reanimated worklet hooks are not memoized by React Compiler; make them stable explicitly and memoize the list wrapper. Use `useDeferredValue` for non-urgent list props during animations.
