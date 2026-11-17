# RainbowImage - unified image component for faster image loads
- **Repository:** rainbow-me/rainbow
- **PR Link:** <https://github.com/rainbow-me/rainbow/pull/6653>
- **Merged Date:** 2025-06-20
- **Category:** Images

## Problem
NFT/collectible images on the home screen used the stock React Native `Image`, which is slower than native image libraries. The PR gives no measurements.

## Solution
Add a `RainbowImage` component that picks the fastest image implementation per URL: `faster-image` for png/jpg/jpeg, `fast-image` for bmp/webp/gif/avif, and RN `Image` for everything else (svg, data: URLs). The format is detected from the URL (including imgix `fm=` params) and memoized. It also exposes `RainbowImage.preload`. `UniqueTokenImage` is switched to it. Caveat from the PR: an upgrade to faster-image 1.7 was reverted because of a reported memory issue; a later move to `expo-image` is suggested.

## Code Pattern (Before vs. After)
```tsx
// Before
<Image onError={onError} onLoad={onLoad} source={{ uri: imageUrl }} style={StyleSheet.absoluteFill} />
{!isLoaded && lowResImageUrl && <Image source={{ uri: lowResImageUrl }} style={StyleSheet.absoluteFill} />}

// After
<RainbowImage onError={onError} onSuccess={onLoad} source={{ url: imageUrl }} style={StyleSheet.absoluteFillObject} />
{!isLoaded && lowResImageUrl && <RainbowImage source={{ url: lowResImageUrl }} style={StyleSheet.absoluteFillObject} />}

// RainbowImage.tsx (handler selection)
const getHandlerFromType = (extension: DetectedImageExtension): ImageHandler => {
  if (extension === 'png' || extension === 'jpg' || extension === 'jpeg') return 'faster-image';
  if (extension === 'bmp' || extension === 'webp' || extension === 'gif' || extension === 'avif') return 'fast-image';
  return 'image';
};
```

## Takeaway
Use a native image library with disk and memory caching for remote images, with a fallback to RN `Image` for formats it cannot handle.
