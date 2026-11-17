# chore: optimize android abi
- **Repository:** OneKeyHQ/app-monorepo
- **PR Link:** <https://github.com/OneKeyHQ/app-monorepo/pull/7777>
- **Merged Date:** 2025-07-15
- **Category:** App Size

## Problem
The Android build compiled native code for `arm64-v8a` and `x86` by default and packaged every ABI the dependencies provided, which grows the APK and lengthens native builds. The PR has no description beyond an auto-generated summary and reports no size numbers.

## Solution
- Release builds now default `reactNativeArchitectures` to `arm64-v8a` only (previously `arm64-v8a,x86`), so only 64-bit ARM native libraries are built for React Native itself.
- Debug builds get an explicit `ndk.abiFilters` of `arm64-v8a` and `x86_64`, so packaged `.so` files are limited to a real device and the 64-bit emulator.
- Caveat: x86 and 32-bit ARM devices are no longer supported by these builds. The default can be overridden with `-PreactNativeArchitectures=...`.

## Code Pattern (Before vs. After)
```properties
# Before (gradle.properties)
reactNativeArchitectures=arm64-v8a,x86

# After
# Default for release builds (safe default)
reactNativeArchitectures=arm64-v8a
```

```groovy
// After (app/build.gradle)
debug {
    resValue "string", "app_name", "OneKey Dev"
    signingConfig signingConfigs.debug
    ndk {
        abiFilters "arm64-v8a", "x86_64"
    }
}
```

## Takeaway
Ship only the ABIs your user base needs; every extra ABI duplicates all native libraries in the package.
