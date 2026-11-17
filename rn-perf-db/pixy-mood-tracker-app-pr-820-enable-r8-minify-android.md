# build(android): enable R8 minify and resource shrinking
- **Repository:** mrzmyr/pixy-mood-tracker-app
- **PR Link:** <https://github.com/mrzmyr/pixy-mood-tracker-app/pull/820>
- **Merged Date:** 2026-10-18
- **Category:** App Size

## Problem
Android release builds had R8 code shrinking and resource shrinking turned off, so unused Java/Kotlin code from all dependencies shipped in the dex files.

## Solution
Enable both through `expo-build-properties` in `app.json`. No extra keep rules were needed: R8 produced no `missing_rules.txt`, and library consumer rules plus the existing `proguard-rules.pro` (reanimated, turbomodule) were enough.
- Measured on a release APK (arm64-v8a, same commit and JS bundle): APK 72.1 MB → 59.9 MB (-12.3 MB, -17%); dex compressed 20.5 MB → 9.0 MB; dex uncompressed 57.7 MB → 23.2 MB; native libs and JS assets unchanged.
- Smoke test on an Android API 35 emulator: cold start `Displayed ... +3s435ms`, key flows rendered, and logcat showed no `ClassNotFoundException`, `NoSuchMethodError` or fatal exceptions.

## Code Pattern (Before vs. After)
```json
// Before (app.json)
"android": {
  "minSdkVersion": 24
}

// After
"android": {
  "minSdkVersion": 24,
  "enableMinifyInReleaseBuilds": true,
  "enableShrinkResourcesInReleaseBuilds": true
}
```

## Takeaway
Enable R8 and resource shrinking for release builds, then smoke test release builds for reflection-related crashes before shipping.
