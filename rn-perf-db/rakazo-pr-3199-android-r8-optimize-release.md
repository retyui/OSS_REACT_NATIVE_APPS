# Optimize Android release code for Google Play
- **Repository:** elie222/rakazo
- **PR Link:** <https://github.com/elie222/rakazo/pull/3199>
- **Merged Date:** 2026-11-04
- **Category:** App Size (+ Native)

## Problem
Expo's generated Android project leaves R8 off, so release bundles shipped unshrunk, unoptimized code. Google Play's Android vitals measured about 1% code optimization and warned that Play will require at least 25% from February 2027. The PR reports no size or runtime numbers.

## Solution
- A config plugin (`with-android-r8`) sets `android.enableMinifyInReleaseBuilds=true` in gradle properties.
- It also swaps the template's `proguard-android.txt` (which contains `-dontoptimize`) for `proguard-android-optimize.txt`; with minification alone R8 still skipped optimization.
- The plugin throws if Expo's `build.gradle` template no longer names the default rules file, and a test reads the pinned Expo template so a version bump fails in CI.
- No new keep rules were added (native modules do not use reflection and ship consumer rules). Resource shrinking stays off because React Native looks up some resources by name. Bundles carry the mapping file for Play crash deobfuscation.

## Code Pattern (Before vs. After)
```js
// Before: Expo template default (R8 off, optimization disabled)
// android.enableMinifyInReleaseBuilds not set
// proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"

// After: apps/mobile/plugins/with-android-r8.js
const MINIFY = "android.enableMinifyInReleaseBuilds";
const DEFAULT_RULES = 'getDefaultProguardFile("proguard-android.txt")';
const OPTIMIZE_RULES = 'getDefaultProguardFile("proguard-android-optimize.txt")';

function applyOptimizedRules(gradle) {
  if (gradle.includes(OPTIMIZE_RULES)) return gradle;
  if (!gradle.includes(DEFAULT_RULES)) {
    throw new Error("with-android-r8: app/build.gradle no longer names the default ProGuard file.");
  }
  return gradle.replace(DEFAULT_RULES, OPTIMIZE_RULES);
}

function withAndroidR8(config) {
  config = withGradleProperties(config, (cfg) => {
    cfg.modResults.push({ type: "property", key: MINIFY, value: "true" });
    return cfg;
  });
  return withAppBuildGradle(config, (cfg) => {
    cfg.modResults.contents = applyOptimizedRules(cfg.modResults.contents);
    return cfg;
  });
}
```

## Takeaway
Enabling R8 on an Expo app needs both the minify property and the optimize ProGuard defaults; guard the template patch with a test.
