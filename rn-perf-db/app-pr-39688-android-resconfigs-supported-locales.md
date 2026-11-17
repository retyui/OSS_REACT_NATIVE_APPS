# [Audit][Implementation] - Use only supported locales to reduce the APK size
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/39688>
- **Merged Date:** 2024-04-10
- **Category:** App Size

## Problem
Android dependencies (google-play-services, material, etc.) ship string resources for dozens of locales, all of which end up in `resources.arsc`, although Expensify only supports `en` and `es`. The PR reports `resources.arsc` dropping from 3.4 MB to 1 MB raw, and total APK size going from 103.1 MB to 100.8 MB (download size 99.3 MB to 98.7 MB), benchmarked on top of a related PR.

## Solution
Declare the supported languages with `resConfigs` in `defaultConfig` so the Android build tools strip every other locale's resources, including those from third-party libraries. Android only; the languages must be declared explicitly, otherwise unsupported variants are not removed. New locales must be added to this list when the app adds them.

## Code Pattern (Before vs. After)
```groovy
// Before
defaultConfig {
    multiDexEnabled rootProject.ext.multiDexEnabled
    versionCode 1001046105
    versionName "1.4.61-5"
}

// After
defaultConfig {
    multiDexEnabled rootProject.ext.multiDexEnabled
    versionCode 1001046105
    versionName "1.4.61-5"
    // Supported language variants must be declared here to avoid from being removed during the compilation.
    // This also helps us to not include unnecessary language variants in the APK.
    resConfigs "en", "es"
}
```
