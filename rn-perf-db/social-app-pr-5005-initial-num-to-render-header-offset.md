# Profile screen performance tweak - Adjust initial num to render based on header height
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/5005>
- **Merged Date:** 2024-08-29
- **Category:** Lists

## Problem
`initialNumToRender` was derived only from window height, ignoring the profile header, which often covers half the screen or more. The profile tab therefore rendered about 9 posts initially when far fewer were visible, slowing tab switches and navigation pushes. The PR says that accounting for the header drops the initial render on the author's profile to four posts and cuts initial render time by around half (screenshots only).

## Solution
- Rework `useInitialNumToRender` to take `{minItemHeight, screenHeightOffset}` and to subtract the offset, the top safe-area inset and the bottom bar height from the window height (using `useWindowDimensions` so it also reacts to size changes).
- The profile posts feed passes `screenHeightOffset: headerHeight` and forwards the result through a new `initialNumToRender` prop on `Feed`, used only for the `posts_and_author_threads` feed.
- `Feed` falls back to its existing value when no override is given; the Messages list call was updated to the new object signature.

## Code Pattern (Before vs. After)
```ts
// Before
export function useInitialNumToRender(minItemHeight: number = MIN_POST_HEIGHT) {
  return React.useMemo(() => {
    const screenHeight = Dimensions.get('window').height
    return Math.ceil(screenHeight / minItemHeight) + 1
  }, [minItemHeight])
}

// After
export function useInitialNumToRender({
  minItemHeight = MIN_POST_HEIGHT,
  screenHeightOffset = 0,
}: {minItemHeight?: number; screenHeightOffset?: number} = {}) {
  const {height: screenHeight} = useWindowDimensions()
  const {top: topInset} = useSafeAreaInsets()
  const bottomBarHeight = useBottomBarOffset()

  const finalHeight =
    screenHeight - screenHeightOffset - topInset - bottomBarHeight
  return Math.floor(finalHeight / minItemHeight) + 1
}
```
```tsx
// Profile Feed section
const adjustedInitialNumToRender = useInitialNumToRender({
  screenHeightOffset: headerHeight,
})
// ...
initialNumToRender={shouldUseAdjustedNumToRender ? adjustedInitialNumToRender : undefined}
```

## Takeaway
Compute `initialNumToRender` from the space actually available to list items (minus headers, insets and bars), not from the full window height.
