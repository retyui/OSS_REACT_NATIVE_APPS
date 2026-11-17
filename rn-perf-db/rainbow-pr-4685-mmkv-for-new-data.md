# perf: use mmkv for new data
- **Repository:** rainbow-me/rainbow
- **PR Link:** <https://github.com/rainbow-me/rainbow/pull/4685>
- **Merged Date:** 2023-03-09
- **Category:** Storage & IO

## Problem
Local persistence used AsyncStorage, which the PR calls orders of magnitude slower than MMKV (citing the public StorageBenchmark; no app-specific numbers given). A full migration of dynamic keys at app start would be costly.

## Solution
A gradual, lazy migration: new writes and reads go to MMKV, and old data is moved on first read.
- New `legacy` storage (MMKV, same `global` id as device data) whose `get()` is async: if the key is missing in MMKV it reads the old AsyncStorage value, writes it to MMKV, then removes the old entry.
- `saveLocal`/`getLocal` now use `legacy`; the old implementations are kept as `deprecatedSaveLocal/GetLocal/RemoveLocal`.
- No up-front migration pass is needed on app start. The PR notes upgrading from an old build should not cause data loss.

## Code Pattern (Before vs. After)
```ts
// Before
export const getLocal = async (key = '', version = defaultVersion) => {
  const result = await storage.load({ key, version /* ... */ });
  // ...
};

// After
export const getLocal = async (key = '') => {
  return await legacy.get([key]);
};

// legacy.ts
const res = this.store.getString(key);
if (!res) {
  const legacyValue = await deprecatedGetLocal(key); // get old value from AsyncStorage
  this.set(scopes, legacyValue); // set first
  deprecatedRemoveLocal(key); // then remove if successful
  return this.get(scopes); // continue as normal
}
return JSON.parse(res).data;
```

## Takeaway
To move off AsyncStorage without a blocking startup migration, migrate each key lazily on first read and keep the old accessor as a deprecated fallback.
