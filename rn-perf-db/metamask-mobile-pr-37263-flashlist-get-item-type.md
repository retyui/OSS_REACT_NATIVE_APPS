# perf: partition Explore Search FlashList recycling pools
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/37263>
- **Merged Date:** 2026-10-08
- **Category:** Lists

## Problem
Explore Search flattens headers, loading skeletons and result rows into a single FlashList v2, but the list did not declare item types, so heterogeneous rows could share one recycling pool. The PR states the FPS improvement was not yet measured on Android and rests on static code evidence.

## Solution
Add `getItemType={(item) => item.type}` so FlashList keeps separate recycling pools per row type and avoids cross-type layout re-measurement during scrolling and tab switches. It is a one-line prop change with no data or UI behavior change.

## Code Pattern (Before vs. After)
```tsx
// Before
<FlashList
  ref={flashListRef}
  data={flatData}
  renderItem={renderFlatItem}
  keyExtractor={keyExtractor}
/>

// After
<FlashList
  ref={flashListRef}
  data={flatData}
  renderItem={renderFlatItem}
  getItemType={(item) => item.type}
  keyExtractor={keyExtractor}
/>
```

## Takeaway
Whenever a FlashList mixes different row layouts, supply `getItemType`.
