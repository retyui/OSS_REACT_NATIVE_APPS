# Work around a startup stall caused by expo-image on low-end Android
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/1801>
- **Merged Date:** 2023-11-03
- **Category:** Startup (+ Images)

## Problem
An apparent issue in `expo-image` blocked the UI thread at startup on low-end Android, so the splash screen stayed up about 5 extra seconds if even one such image rendered. The startup-blocking images were the user's own avatars in the drawer and tab bar. Reported: time to end of splash 8s -> ~3.5s, time to content 13s -> ~10.5s (network-bound, harder to measure).

## Solution
- Add a `usePlainRNImage` prop to `UserAvatar` that renders React Native's built-in `Image` instead of the expo-image-based `HighPriorityImage`.
- Enable it only for the avatars in `Drawer` and `BottomBar`, which render at startup.
- Labeled a temporary workaround; the proper fix is to report and fix it upstream in Expo.

## Code Pattern (Before vs. After)
```tsx
// Before
<HighPriorityImage
  testID="userAvatarImage"
  style={aviStyle}
  contentFit="cover"
  source={{uri: avatar}}
  blurRadius={moderation?.blur ? BLUR_AMOUNT : 0}
/>

// After
{usePlainRNImage ? (
  <Image
    accessibilityIgnoresInvertColors
    testID="userAvatarImage"
    style={aviStyle}
    resizeMode="cover"
    source={{uri: avatar}}
    blurRadius={moderation?.blur ? BLUR_AMOUNT : 0}
  />
) : (
  <HighPriorityImage /* ...same props as before */ />
)}
```
```tsx
// Drawer.tsx / BottomBar.tsx
<UserAvatar size={80} avatar={store.me.avatar} usePlainRNImage={true} />
```
