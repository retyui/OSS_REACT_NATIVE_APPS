# chore: do not disable virtualization on new architecture
- **Repository:** artsy/eigen
- **PR Link:** <https://github.com/artsy/eigen/pull/12831>
- **Merged Date:** 2025-10-10
- **Category:** Lists (+ Memory)

## Problem
Home view horizontal rails had `disableVirtualization` set as a workaround for broken virtualization in nested FlatLists. That hack renders every item up front. The PR states it is not needed on the new architecture and that, with it set, it "breaks memory and CPU". No measurements are included.

## Solution
- Add an `isNewArchitectureEnabled` helper that checks for the global `nativeFabricUIManager` (present only under Fabric).
- Replace the hard-coded `disableVirtualization` on the rails (ArtworkRail, ArticlesRail, activity, artists, auction results, fairs, FairCollections) with `disableVirtualization={!isNewArchitectureEnabled}`, so virtualization stays on under the new architecture and the workaround remains on the old one.
- Tested on both architectures per the PR. As merged, `ShowsRail` uses the non-negated `isNewArchitectureEnabled`, which looks inverted relative to the other rails.

## Code Pattern (Before vs. After)
```tsx
// Before
<FlatList
  horizontal
  // This is required to avoid broken virtualization on nested flatlists
  disableVirtualization
  keyExtractor={(item: Artwork) => item.internalID}
/>

// After
// src/app/utils/isNewArchitectureEnabled.ts
// @ts-expect-error nativeFabricUIManager is only available in the new architecture and isn't typed yet
export const isNewArchitectureEnabled = !!global.nativeFabricUIManager

<FlatList
  horizontal
  disableVirtualization={!isNewArchitectureEnabled}
  keyExtractor={(item: Artwork) => item.internalID}
/>
```

## Takeaway
Re-check list virtualization workarounds after architecture upgrades, and gate them on the runtime that needs them instead of disabling virtualization everywhere.
