# perf: stop shipping duplicate brand fonts through expo-font
- **Repository:** OneKeyHQ/app-monorepo
- **PR Link:** <https://github.com/OneKeyHQ/app-monorepo/pull/14418>
- **Merged Date:** 2026-11-10
- **Category:** App Size (+ Startup)

## Problem
The native brand typeface was compiled into the app twice. The JS side called `useFonts({...require(ttf)})`, so Metro shipped the 6 brand and mono `.ttf` files as Android `res/raw` assets. The native side already copied the same files to `android/app/src/main/assets/fonts` and to the iOS `UIAppFonts` folder. A scan of the 6.7.0 APK found 6 byte-identical ttf pairs, 1.35 MiB raw and about 0.66 MiB inside the APK. The PR expects about -0.52 MiB of APK size and notes that post-build size was not verified.

## Solution
- `useLoadCustomFonts.ts` is deleted, and `FontProvider` (which blocked rendering until `useFonts` finished) is removed, so native no longer requires the ttf files. Fonts are still registered by `ReactFontManager` (Android) and `UIAppFonts` (iOS).
- Dropped the direct `expo-font` dependency, its `app.json` plugin entry and its `dev-vendor.config.js` entry, and added `expo-font` to the Expo autolinking `exclude` list for Android and iOS. The exclude is what actually removes the native module from the build, because `expo` still depends on the package.
- The font lint check now validates the native font slots directly and fails if a JS-side `expo-font` loader returns.
- Caveat: one brand font is still bundled for the chart typeface, which loads it through `Image.resolveAssetSource(require(ttf))` (about 0.14 MiB left to dedupe in a follow-up). Real-device font rendering still needed QA.

## Code Pattern (Before vs. After)
```tsx
// Before (useLoadCustomFonts.ts): Metro bundles every ttf into the APK
const customFont = {
  [BRAND_FONT_NATIVE_FAMILY.regular]: require('../fonts/brand/BrandSans-Regular.ttf'),
  [BRAND_FONT_NATIVE_FAMILY.medium]: require('../fonts/brand/BrandSans-Medium.ttf'),
  'GeistMono-Regular': require('../fonts/GeistMono-Regular.ttf'),
  // ...
};
export default function useLoadCustomFonts() {
  return useFonts(customFont);
}

// ...and a render gate in the provider:
const [loaded, error] = useLoadCustomFonts();
if (platformEnv.isNative || platformEnv.isRuntimeBrowser) {
  return loaded || error ? children : null;
}

// After: provider renders children directly; no JS font loading
<SettingConfigContext.Provider value={providerValue}>
  <SafeAreaProvider>
    <MemoizedTamaguiProvider config={config} defaultTheme={theme}>
      {children}
    </MemoizedTamaguiProvider>
  </SafeAreaProvider>
</SettingConfigContext.Provider>
```

```json
// After (apps/mobile/package.json): exclude the native module from autolinking
"android": { "exclude": ["expo-image", "expo-font"] },
"ios":     { "exclude": ["expo-image", "expo-font"] }
```

## Takeaway
If fonts are already registered natively, loading them again through `expo-font` duplicates the files in the package and adds a render gate. Register fonts in one place only.
