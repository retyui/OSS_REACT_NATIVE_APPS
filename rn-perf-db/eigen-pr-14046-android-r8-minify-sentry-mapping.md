# build(android): minify android builds to reduce size with r8
- **Repository:** artsy/eigen
- **PR Link:** <https://github.com/artsy/eigen/pull/14046>
- **Merged Date:** 2026-09-29
- **Category:** App Size

## Problem
Android release builds were not minified, even though the build already read `android.enableMinifyInReleaseBuilds`; it was never set. Measured with `bundletool get-size total` on the same machine, arm64-v8a download size was 56.28-59.41 MiB, with dex at 78.88 MiB uncompressed across 8 dex files.

## Solution
- Set `android.enableMinifyInReleaseBuilds=true` and `android.enableShrinkResourcesInReleaseBuilds=true` in `gradle.properties`, and wire `shrinkResources` in `build.gradle`.
- Add `-dontwarn` rules for Fresco's unbundled animated-image backend and Stripe's optional push-provisioning SDK, which otherwise fail R8 with missing classes.
- Apply the Sentry Android Gradle plugin (6.22.0) to upload the R8 `mapping.txt` so native crashes stay readable; upload is opt-in via `SENTRY_UPLOAD_PROGUARD_MAPPING` and only set by deploy lanes.
- Raise Gradle JVM heap from 3g to 4g because R8 ran out of memory at 3g in CI; 6g starved the runner.
- Disable minification for Maestro and verification builds, which are never shipped.
- Result per the PR: arm64-v8a download 40.39-43.43 MiB (-15.89 MiB, -28.2%), dex 30.98 MiB (-60.7%), dex files 8 to 5; native libs unchanged.

## Code Pattern (Before vs. After)
```groovy
// Before (android/app/build.gradle)
minifyEnabled enableMinifyInReleaseBuilds
proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"

// After
def enableShrinkResourcesInReleaseBuilds = (findProperty('android.enableShrinkResourcesInReleaseBuilds') ?: false).toBoolean()
// ...
minifyEnabled enableMinifyInReleaseBuilds
shrinkResources enableShrinkResourcesInReleaseBuilds
proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
```

```properties
# After (android/gradle.properties)
org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=1024m
android.enableMinifyInReleaseBuilds=true
android.enableShrinkResourcesInReleaseBuilds=true
```

```text
# After (proguard-rules.pro)
-keepattributes SourceFile,LineNumberTable
-renamesourcefileattribute SourceFile
-dontwarn com.facebook.imagepipeline.animated.factory.AnimatedFactoryImpl
-dontwarn com.stripe.android.pushProvisioning.**
```

## Takeaway
R8 and resource shrinking mostly shrink dex, not native libs; wire crash-symbol mapping upload before shipping, and budget extra Gradle heap for CI.
