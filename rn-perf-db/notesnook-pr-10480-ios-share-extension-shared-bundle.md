# Ios app size reduction
- **Repository:** streetwriters/notesnook
- **PR Link:** <https://github.com/streetwriters/notesnook/pull/10480>
- **Merged Date:** 2026-10-15
- **Category:** App Size

## Problem
The iOS share extension ("Make Note") shipped its own copy of the JS bundle and its own Inter font files, duplicating what the main app already contains. Code comments in the diff put the duplicate bundle at about 10MB and the duplicate fonts at about 1.6MB. The PR body only says it reduces iOS app size by sharing code between app and extension.

## Solution
- In release builds, the extension's `bundleURL` points at the containing app's `main.jsbundle` (two directories up from `.../Notesnook.app/PlugIns/Make Note.appex`) instead of a bundled copy, with a fallback to a local copy. Debug still uses the Metro URL.
- The app's `index.js` already registers `NotesnookShare`, so the shared bundle works, but module-scope side effects (`BackgroundSync`, `Notifications.init`) now run only when `NativeModules.ShareViewController` is absent, i.e. not in the extension. The old `index.ext.js` was deleted.
- Inter fonts were removed from the extension's `UIAppFonts` and Resources, and registered at runtime from the app bundle with `CTFontManagerRegisterFontsForURL` (process scope). Unused Inter weights in `packages/editor-mobile/public/fonts` were also deleted.
- Risk: the extension now depends on the app bundle layout and on gating any new app-only startup code.

## Code Pattern (Before vs. After)
```objc
// Before
- (NSURL *)bundleURL
{
  return [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index"];
}
```
```objc
// After
- (NSURL *)bundleURL
{
#if DEBUG
  return [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index"];
#else
  NSURL *bundleURL = [NNContainingAppBundleURL()
                      URLByAppendingPathComponent:@"main.jsbundle"];
  if ([[NSFileManager defaultManager] fileExistsAtPath:bundleURL.path]) {
    return bundleURL;
  }
  return [[NSBundle mainBundle] URLForResource:@"main" withExtension:@"jsbundle"];
#endif
}
```
```js
// index.js (After)
const IS_SHARE_EXTENSION = (() => {
  try { return !!NativeModules.ShareViewController; } catch (e) { return false; }
})();

if (!IS_SHARE_EXTENSION) {
  BackgroundSync.registerHeadlessTask();
  BackgroundSync.start();
  Notifications.init();
}
```

## Takeaway
App extensions can load the host app's JS bundle and fonts instead of embedding copies; gate app-only side effects in the shared entry file.
