# fix(ios): cap SDWebImage memory cache to reduce OOM crashes
- **Repository:** artsy/eigen
- **PR Link:** <https://github.com/artsy/eigen/pull/14451>
- **Merged Date:** 2026-11-11
- **Category:** Memory (+ Images)

## Problem
SDWebImage (used by FastImage) keeps every decoded image in memory and only evicts under system pressure. Long scrolls through artwork grids kept growing the app until iOS killed it with out-of-memory crashes, mostly on iPad. Measured on an iPad simulator scrolling a ~4,850-work grid, peak memory went from 3,211 MB to 1,806 MB (-44%).

## Solution
- Cap `SDImageCache.shared.config.maxMemoryCost` at 250 MB in the app delegate at launch.
- Evicted images are re-decoded from the disk cache, not re-downloaded, so there is no visual change; scrolling back up re-renders from disk.
- Required importing `SDImageCache.h` in the bridging header.

## Code Pattern (Before vs. After)
```swift
// Before
helper?.setup(launchOptions: launchOptions)
// (no memory cache limit; SDWebImage evicts only on system memory pressure)

// After
helper?.setup(launchOptions: launchOptions)
configureImageMemoryCache()
// ...
private func configureImageMemoryCache() {
    SDImageCache.shared.config.maxMemoryCost = 250 * 1024 * 1024 // bytes of decoded pixels
}
```
```objc
// Artsy-Bridging-Header.h
#import <SDWebImage/SDImageCache.h>
```

## Takeaway
Set an explicit in-memory cost limit on native image caches for apps that show very long image grids.
