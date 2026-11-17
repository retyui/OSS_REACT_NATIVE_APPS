# APP-3022: enable R8 minification for Android releases
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/11647>
- **Merged Date:** 2026-09-23
- **Category:** App Size (+ Native)

## Problem
Android release builds shipped without R8 code shrinking/obfuscation. For the arm64 release APK, the PR reports Android download size dropping from 75.27 MB to 62.34 MB (12.93 MB, 17.2% smaller) once R8 is enabled.

## Solution
- Set `enableMinifyInReleaseBuilds: true` through `expo-build-properties`.
- Enable the Sentry Android Gradle plugin (`experimental_android.enableAndroidGradlePlugin`) so ProGuard mappings and native symbols are uploaded for Sentry-enabled release builds.
- Resource shrinking is left off because it saved only 0.09 MB in testing.
- Google Play gets the deobfuscation mapping from the AAB (verified identical to Gradle's `mapping.txt`). The test plan calls for smoke-testing launch, timeline, posting and media selection, since minification can break reflection-based code.

## Code Pattern (Before vs. After)
```js
// Before (app.config.js)
targetSdkVersion: 36,
buildToolsVersion: '36.0.0',
buildReactNativeFromSource: IS_PRODUCTION,

// After
targetSdkVersion: 36,
buildToolsVersion: '36.0.0',
buildReactNativeFromSource: IS_PRODUCTION,
enableMinifyInReleaseBuilds: true,

// ...and in the Sentry plugin options:
{
  organization: 'blueskyweb',
  project: 'app',
  url: 'https://sentry.io',
  experimental_android: {
    enableAndroidGradlePlugin: true,
  },
}
```

## Takeaway
Expo's generated Android project leaves R8 off; enable it via `expo-build-properties` and make sure mapping files reach your crash reporter.
