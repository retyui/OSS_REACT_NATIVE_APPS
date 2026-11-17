# [LWM] fix(lwm): drop R8 optimization to fix custom lock screen (LIVE-37084)
- **Repository:** LedgerHQ/ledger-live
- **PR Link:** <https://github.com/LedgerHQ/ledger-live/pull/21648>
- **Merged Date:** 2026-09-08
- **Category:** App Size

## Problem
With R8 enabled, the custom lock screen flow failed in minified builds with expo-modules-core's `Call to function 'Context.resize' has been rejected`. Bisecting five signed `stagingRelease` builds on a physical device showed the R8 optimizer (not shrinking or obfuscation) was the cause: keep rules for `expo.modules.imagemanipulator.**` or `kotlin.**` did not help, while `-dontoptimize` did. The PR reports dex sizes of 47.7 MB / 6 files unminified, 18.3 MB / 3 files with R8 and optimizer disabled, and 12.9 MB / 2 files with the optimizer on.

## Solution
- Switch from `proguard-android-optimize.txt` to AGP's `proguard-android.txt`. The two default files differ by two flags: this swaps `-allowaccessmodification` for `-dontoptimize`.
- Shrinking and obfuscation stay on, so most of the size win survives (about 5 MB of dex is given up versus the optimizer).
- Caveats from the PR: the exact optimization pass that breaks Expo is not isolated, and CI could not validate the change at the time because R8 was temporarily off, so the manual bisection is the only evidence.

## Code Pattern (Before vs. After)
```groovy
// Before
release {
    minifyEnabled enableProguardInReleaseBuilds
    proguardFiles getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro"
    matchingFallbacks = ['debug']
}

// After
release {
    minifyEnabled enableProguardInReleaseBuilds
    proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
    matchingFallbacks = ['debug']
}
```

## Takeaway
If R8 breaks an Expo/JSI module at runtime, try `-dontoptimize` before writing broad keep rules: you keep shrinking and obfuscation (most of the dex savings) and lose only the optimizer.
