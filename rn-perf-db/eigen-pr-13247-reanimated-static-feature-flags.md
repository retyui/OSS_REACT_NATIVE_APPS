# chore: upgrade reanimated and set performance flags
- **Repository:** artsy/eigen
- **PR Link:** <https://github.com/artsy/eigen/pull/13247>
- **Merged Date:** 2026-02-19
- **Category:** Animations (+ Native)

## Problem
On the New Architecture, Reanimated by default applies animated non-layout props (transform, opacity, colors) by committing to the React shadow tree. Under load this competes with React commits and can drop frames. Artsy was also still on Reanimated 3.19.5. The PR body only links to an internal PR and reports no numbers.

## Solution
- Upgrade `react-native-reanimated` from 3.19.5 to 4.2.1, which adds `react-native-worklets`, and fix the code that broke (a bottom-sheet patch, artist screens, tests).
- Turn on Reanimated 4 **static feature flags** in `package.json`. They are read at build time, so iOS needs a `pod install`.
  - `ANDROID_SYNCHRONOUSLY_UPDATE_UI_PROPS` / `IOS_SYNCHRONOUSLY_UPDATE_UI_PROPS`: non-layout props are applied straight to the native view, without a shadow-tree commit.
  - `USE_COMMIT_HOOK_ONLY_FOR_REACT_COMMITS`: Reanimated's commit hook runs only for React commits, not for its own.
  - `DISABLE_COMMIT_PAUSING_MECHANISM`: meant to be used together with RN's experimental `preventShadowTreeCommitExhaustion`.
- Set `reactNativeReleaseLevel=experimental` in `gradle.properties`, `Info.plist` and the `expo-build-properties` options in `app.json`. This turns on RN's experimental feature-flag set, which includes the shadow-tree commit-exhaustion guard that pairs with the flag above.
- Risks: the experimental release level turns on unstable RN features, and the major Reanimated upgrade needed follow-up fixes. A later Eigen change (#14046) shows the Android release level back at `stable`. Measure before copying this.

## Code Pattern (Before vs. After)
```json
// Before (package.json)
// (no "reanimated" key)

// After (package.json)
"reanimated": {
  "staticFeatureFlags": {
    "USE_COMMIT_HOOK_ONLY_FOR_REACT_COMMITS": true,
    "DISABLE_COMMIT_PAUSING_MECHANISM": true,
    "ANDROID_SYNCHRONOUSLY_UPDATE_UI_PROPS": true,
    "IOS_SYNCHRONOUSLY_UPDATE_UI_PROPS": true
  }
},
```

```properties
# After (android/gradle.properties)
# Specify the release level for the React Native build.
reactNativeReleaseLevel=experimental
```

```xml
<!-- After (ios/Artsy/App_Resources/Info.plist) -->
<key>ReactNativeReleaseLevel</key>
<string>experimental</string>
```

## Takeaway
On Reanimated 4 with the New Architecture, the `*_SYNCHRONOUSLY_UPDATE_UI_PROPS` and `USE_COMMIT_HOOK_ONLY_FOR_REACT_COMMITS` flags are the cheap win. Turn on `DISABLE_COMMIT_PAUSING_MECHANISM` only together with RN's `preventShadowTreeCommitExhaustion` (the experimental release level).
