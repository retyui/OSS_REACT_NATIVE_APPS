# Downscale expo-image images off the main thread on iOS
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/12055>
- **Merged Date:** 2026-10-21
- **Category:** Images (+ Native)

## Problem
When a loaded image is larger than its view, expo-image redraws it at display size (`resize(image:toSize:scale:)`, a `UIGraphicsImageRenderer` draw) inside the load completion handler on the main thread. In Bluesky's feeds this cost 50-140 ms of main-thread time per screen load or scroll run. Measured on iPhone 14 Pro (Release): cold-open main-thread image work 59/100/87 ms (Home/profile/thread) to 9/11/29 ms; Home scrolling 131 to 31 ms; profile scrolling 112 to 29 ms. Total main-thread CPU while scrolling did not change measurably.

## Solution
- A `patch-package` patch to `expo-image` (`ImageView.swift`) runs the resize on `DispatchQueue.global(qos: .userInitiated)` and renders the result back on the main queue.
- A `loadGeneration` counter, bumped in `cancelPendingOperation()` (new source, recycling, dealloc), drops results for loads the view no longer shows.
- Animated images and images that do not need downscaling use the old path.
- Trade-off: a downscaled image appears a few ms later; until then the placeholder or previous image shows. The patch should be removed if upstream moves downscaling off the main thread.

## Code Pattern (Before vs. After)
```swift
// Before
let image = processImage(image, idealSize: imageIdealSize, scale: scale)
applyContentPosition(contentSize: imageLayoutSize, containerSize: frame.size)
renderSourceImage(image)

// After
if allowDownscaling && !bounds.isEmpty && !image.sd_isAnimated
  && shouldDownscale(image: image, toSize: imageIdealSize, scale: scale) {
  sdImageView.animationTransformer = nil
  let generation = loadGeneration
  let layoutSize = imageLayoutSize
  DispatchQueue.global(qos: .userInitiated).async { [weak self] in
    let resized = resize(image: image, toSize: imageIdealSize, scale: scale)
    DispatchQueue.main.async {
      guard let self, self.loadGeneration == generation else { return }
      self.applyContentPosition(contentSize: layoutSize, containerSize: self.frame.size)
      self.renderSourceImage(resized)
    }
  }
  return
}
// ...
func cancelPendingOperation() {
  pendingOperation?.cancel()
  pendingOperation = nil
  loadGeneration &+= 1
}
```

## Takeaway
Move CPU-heavy image decode/resize work out of load callbacks that run on the main thread, and guard the async result with a generation token to handle view recycling.
