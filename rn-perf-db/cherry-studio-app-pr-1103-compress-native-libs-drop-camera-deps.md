# perf(android-build): reduce release APK size
- **Repository:** CherryHQ/cherry-studio-app
- **PR Link:** <https://github.com/CherryHQ/cherry-studio-app/pull/1103>
- **Merged Date:** 2026-09-29
- **Category:** App Size

## Problem
Android release APKs shipped uncompressed native libraries and carried two unused dependencies, `react-native-vision-camera` and `react-native-nitro-image`. The PR measured the production APK (versionCode 89) at 125.6 MiB. It reports 7.0 MiB of `.so` files plus Java code from the two unused packages, and 45 native libraries that took 66.1 MiB stored versus 21.1 MiB deflated (-68%). It expects roughly 78 MiB for the production APK, but a release build on a device had not been verified when the PR was written.

## Solution
- Remove the two unused dependencies. QR pairing uses `expo-camera`.
- Set `useLegacyPackaging` through the `expo-build-properties` plugin so native `.so` files are compressed in the direct-download APK profiles (`preview`, `production`).
- The Google Play AAB profile sets `ANDROID_COMPRESS_NATIVE_LIBS=false`, because Play compresses the download itself and compressed libraries in an AAB only slow installation and double on-device storage.
- Give the EAS release Gradle commands `-Dorg.gradle.jvmargs=-Xmx4096m`. This has no size effect but removes a Metaspace cap that broke local builds.
- R8 was deliberately left off, to keep stack traces readable without a mapping upload. Hermes bytecode stays uncompressed. Compressed libraries are extracted at install time, so this does not reduce installed storage or speed up startup.

## Code Pattern (Before vs. After)
```ts
// Before (app.config.ts): expo-build-properties used as-is, native libs stored uncompressed

// After
if (Array.isArray(plugin) && plugin[0] === 'expo-build-properties') {
  return [
    plugin[0],
    {
      ...plugin[1],
      android: {
        ...plugin[1].android,
        // Compressed native libraries only help direct APK downloads; the Google Play profile opts out.
        useLegacyPackaging:
          profile !== 'development' && process.env.ANDROID_COMPRESS_NATIVE_LIBS !== 'false',
      },
    },
  ];
}
```
```json
// eas.json, Before
"gradleCommand": ":app:assembleRelease -PreactNativeArchitectures=arm64-v8a"

// eas.json, After (production-google-play also sets "ANDROID_COMPRESS_NATIVE_LIBS": "false")
"gradleCommand": ":app:assembleRelease -PreactNativeArchitectures=arm64-v8a -Dorg.gradle.jvmargs=-Xmx4096m"
```

## Takeaway
Audit `package.json` for native modules that nothing imports, and compress native libraries only for APKs distributed directly, not for AABs.
