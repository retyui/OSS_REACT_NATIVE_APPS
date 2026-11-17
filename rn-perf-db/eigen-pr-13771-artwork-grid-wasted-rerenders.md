# perf: reduce wasted re-renders in artwork grids
- **Repository:** artsy/eigen
- **PR Link:** <https://github.com/artsy/eigen/pull/13771>
- **Merged Date:** 2026-07-21
- **Category:** Lists (+ Rendering)

## Problem
Artwork grids (FlashList masonry) re-rendered unnecessarily while scrolling. Each artwork tile mounted a hidden "Create Alert" modal subtree (with its own Relay fragment), and container screens passed fresh objects, functions and inline footer components to the list. With React DevTools on a physical device, per scroll: `CreateArtworkAlertModal` about 230 re-renders to 0, `CreateSavedSearchModal` about 259 to 35, `ArtworkSaveIconWrapper` about 280 to 0. Per pagination, filter-header re-renders went 17 to 0 (Fair), 19 to 0 (Collection), 29 to 13 (Artist). The author says only the cell-level change is measurable as a scroll win.

## Solution
- In `ArtworkGridItem`, render `CreateArtworkAlertModal` only while its "open" flag is true.
- Wrap `ArtworkSaveIconWrapper` in `React.memo` (all props are primitives).
- Across nine grid containers, hoist `keyExtractor` and static styles, `useMemo` the data, header, footer and `contentContainerStyle`, `useCallback` the handlers, and replace inline `ListFooterComponent={() => ...}` with a stable element.
- Wrap the functions returned by `useDismissAlertReminder` in `useCallback`, since otherwise they defeated memoization in `ArtistArtworks`.

## Code Pattern (Before vs. After)
```tsx
// Before
<CreateArtworkAlertModal
  artwork={artwork}
  onClose={() => setShowCreateArtworkAlertModal(false)}
  visible={showCreateArtworkAlertModal}
/>

// After
{!!showCreateArtworkAlertModal && (
  <CreateArtworkAlertModal
    artwork={artwork}
    onClose={() => setShowCreateArtworkAlertModal(false)}
    visible={showCreateArtworkAlertModal}
  />
)}
```

```tsx
// Before
export const ArtworkSaveIconWrapper: React.FC<ArtworkSaveIconWrapperProps> = ({
  isSaved, testID, accessibilityLabel, fill,
}) => { /* ... */ }

// After
export const ArtworkSaveIconWrapper = memo(function ArtworkSaveIconWrapper({
  isSaved, testID, accessibilityLabel, fill,
}: ArtworkSaveIconWrapperProps) { /* ... */ })
```

## Takeaway
In recycled list cells, do not mount hidden modals or fragment-reading subtrees, and memoize leaf components with primitive props; keep list props referentially stable.
