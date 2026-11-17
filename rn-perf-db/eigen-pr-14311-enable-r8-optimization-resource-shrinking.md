# build(android): enable R8 optimization and optimized resource shrinking
- **Repository:** artsy/eigen
- **PR Link:** <https://github.com/artsy/eigen/pull/14311>
- **Merged Date:** 2026-10-12
- **Category:** App Size (+ Startup)

## Problem
The Android release build used the default `proguard-android.txt`, which includes `-dontoptimize`, so R8 ran without code optimization. Play Store tips recommended enabling R8 optimization for better memory and performance. The PR author measured before/after on an old device with Flashlight (two runs), but the numbers are only in screenshots, not in the text.

## Solution
- Switch the release `proguardFiles` to `proguard-android-optimize.txt` so R8 optimizes code.
- Set `android.r8.optimizedResourceShrinking=true` so R8 shrinks code and resources together; resources referenced only from removed code are dropped too. It is opt-in on AGP 8.12/8.13 and the default in AGP 9.
- Follow-ups listed: verify all flows on an Android beta and check that Sentry crashes still report properly (optimization can break reflection-dependent code or obfuscated stack traces).

## Code Pattern (Before vs. After)
```groovy
// Before (android/app/build.gradle)
proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"

// After
proguardFiles getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro"
```
```properties
# After (android/gradle.properties)
android.r8.optimizedResourceShrinking=true
```

## Takeaway
Check that release builds do not inherit `-dontoptimize`; switch to the optimize proguard file and test with keep rules.
