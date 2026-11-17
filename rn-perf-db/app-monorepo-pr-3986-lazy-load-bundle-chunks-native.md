# feat: lazy load bundles by native module in Prod. 
- **Repository:** OneKeyHQ/app-monorepo
- **PR Link:** <https://github.com/OneKeyHQ/app-monorepo/pull/3986>
- **Merged Date:** 2023-12-28
- **Category:** Startup (+ Bundle Size)

## Problem
The mobile app shipped a single large JS bundle that Hermes had to load and execute at launch. Split-bundle support existed only in development, where chunks were fetched from the Metro dev server over HTTP. The PR description is empty and reports no measurements.

## Solution
- Production builds now emit dynamic-import chunks as `.bundle` files and copy them into the iOS and Android native assets (`react-native-copy-asset`).
- A new native `Bundle` module (`RCTBundleModule` on iOS, `BundleModule` on Android) exposes `executeSourceCode(hash)`. iOS memory-maps the file (`NSDataReadingMappedIfSafe`) and runs it with `executeSourceCode:withSourceURL:sync:` on the bridge. Android calls `loadScriptFromAssets` on the catalyst instance.
- The `asyncRequire` template picks the HTTP fetcher in dev and the native loader otherwise. Each chunk appends a small footer that resolves pending promises once it has executed.
- The chunk-to-hash map is simplified so a chunk id maps directly to its hash, and chunks are only evaluated on first `import()`.
- Caveat: it relies on private bridge APIs (`RCTBridge+Private`, `getCatalystInstance`), so it is tied to the RN version in use.

## Code Pattern (Before vs. After)
```js
// Before: every chunk fetched over HTTP and evaluated with Function()
const fetchModule = async (hash) => {
  const url = `http://__METRO_HOST_IP__:8081/async-thunks?hash=${hash}`;
  const response = await fetch(url);
  const text = await response.text();
  return text;
};

// After: production loads chunks from bundled assets via a native module
const { Bundle } = NativeModules;
const fetchNativeModule = (hash) =>
  new Promise((resolve) => {
    const { pendingChunks } = global;
    pendingChunks[hash] = pendingChunks[hash] || [];
    pendingChunks[hash].push(resolve);
    Bundle.executeSourceCode(hash);
  });

const fetchModule =
  '__NODE_ENV__' !== 'production' ? fetchHttpModule : fetchNativeModule;
```

```objc
// iOS native loader
RCT_EXPORT_METHOD(executeSourceCode:(NSString *)hashId)
{
  NSString *path = [NSString stringWithFormat:@"%@/%@.bundle", resourcePath, hashId];
  NSData* data = [NSData dataWithContentsOfFile:path options:NSDataReadingMappedIfSafe error:&error];
  [RCTBridge.currentBridge executeSourceCode:data withSourceURL:[NSURL URLWithString:path] sync:NO];
}
```

```java
// Android native loader
@ReactMethod
public void executeSourceCode(String hashId) {
    this.reactNativeHost.getReactInstanceManager().getCurrentReactContext()
        .getCatalystInstance()
        .loadScriptFromAssets(this.reactContext.getAssets(),
            "assets://custom/" + hashId + ".bundle", false);
}
```

## Takeaway
Splitting rarely used screens into on-demand chunks loaded from local assets keeps the startup bundle small without needing a network fetch in production.
