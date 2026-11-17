# [LWM] perf(lwm): enable R8 on Android release builds (LIVE-28531)
- **Repository:** LedgerHQ/ledger-live
- **PR Link:** <https://github.com/LedgerHQ/ledger-live/pull/21296>
- **Merged Date:** 2026-09-01
- **Category:** App Size

## Problem
The React Native template ships `enableProguardInReleaseBuilds = false` and it was never flipped, so every Android bundle contained unminified dex and Google Play flagged the app as unoptimized. The PR reports APK 152.2 MB -> 139.5 MB and dex 47.7 MB / 6 files -> 12.9 MB / 2 files, at the cost of about 1m55s extra per release build.

## Solution
- Enable `minifyEnabled` (R8) on `release` (inherited by `stagingRelease`) and `shrinkResources`, using `proguard-android-optimize.txt`.
- Add `-keep class com.ledger.live.BuildConfig { *; }`: `react-native-config` reads env via reflection (`Class.forName` + `getDeclaredFields`), so R8 saw no reference and removed the class, leaving every `Config.*` empty at runtime (the app still booted, but Datadog was silently disabled).
- Detox variants get their own rules (`-keep class kotlin.**`, `kotlinx.coroutines.**`) because the instrumentation APK is not minified and links against a stdlib R8 may shrink or rename; Detox's own rules moved off `release`.
- Follow-up noted: upload the R8 `mapping.txt` to Datadog so native stack traces are readable.

## Code Pattern (Before vs. After)
```groovy
// Before
def enableProguardInReleaseBuilds = false
// ...
release {
    minifyEnabled enableProguardInReleaseBuilds
    proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
    proguardFile "${rootProject.projectDir}/../node_modules/detox/android/detox/proguard-rules-app.pro"
}

// After
def enableProguardInReleaseBuilds = true
// ...
release {
    minifyEnabled enableProguardInReleaseBuilds
    shrinkResources enableProguardInReleaseBuilds
    proguardFiles getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro"
}
```

```text
# proguard-rules.pro
-keep class com.ledger.live.BuildConfig { *; }
```

## Takeaway
Turn on R8 and resource shrinking for release builds, and add keep rules for classes reached only via reflection (such as the `BuildConfig` read by `react-native-config`). Verify with a signed release build, since missing keep rules fail silently.
