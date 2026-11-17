# perf(android): enable R8 minification and resource shrinking
- **Repository:** RocketChat/Rocket.Chat.ReactNative
- **PR Link:** <https://github.com/RocketChat/Rocket.Chat.ReactNative/pull/7787>
- **Merged Date:** 2026-10-13
- **Category:** App Size (+ Memory, Startup)

## Problem
Android release builds shipped with minification off (`enableProguardInReleaseBuilds = false`), so all unused Java/Kotlin code and resources went into the APK. On an arm64 release build (Android 14 emulator) the PR measures dex at 50.4 MB across 6 files and an estimated Play download of 50.5 MB, with total PSS after cold start of 181-187 MB.

## Solution
- Turn on `minifyEnabled` and `shrinkResources`, using `proguard-android-optimize.txt`.
- Add `@Keep` to every Gson-serialized model in the notification and VoIP code, and add `proguard-rules.pro` keep rules for classes found by name from native code or reflection (WatermelonDB JSI, which otherwise crashes at launch with `NoSuchMethodError`, plus Install Referrer, `GoogleApiAvailability`, ML Kit barcode scanning, keyboard-controller and react-native-screens members).
- Bump Gson 2.8.9 to 2.13.2 and the Crashlytics Gradle plugin 2.9.0 to 3.0.8, because 2.9.0's mapping upload fails on Gradle 9 with minify on.
- Reported result: dex 13.3 MB in 2 files, Play download estimate 37.8 MB, code PSS ~80 MB to ~58 MB, total PSS 146-159 MB, process start to JS running 371 ms to 346 ms (n=24); first frame change was not significant.
- Risk: reflection and JNI lookups can break silently, so the PR lists a manual release-build test plan (push, E2EE push, VoIP, QR scan).

## Code Pattern (Before vs. After)
```groovy
// Before
def enableProguardInReleaseBuilds = false
// ...
release {
    signingConfig signingConfigs.release
    minifyEnabled enableProguardInReleaseBuilds
    proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
}

// After
release {
    signingConfig signingConfigs.release
    minifyEnabled true
    shrinkResources true
    proguardFiles getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro"
}
```

```text
# proguard-rules.pro: looked up by name from native code or reflection, which R8 cannot trace
-keep class com.nozbe.watermelondb.jsi.** { *; }
-keep class com.android.installreferrer.api.** { *; }
-keep class com.google.android.gms.common.GoogleApiAvailability { *; }
-keepnames class com.google.mlkit.vision.barcode.BarcodeScanning
-keepclassmembers class com.facebook.react.uimanager.JSPointerDispatcher {
    public void handleMotionEvent(...);
}
```

## Takeaway
Enable R8 and resource shrinking on release builds, then test every native-lookup path (JSI, reflection, Gson models) on a release build and keep what R8 cannot trace.
