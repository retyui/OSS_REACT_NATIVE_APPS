# fix(android): skip onSaveInstanceState under low memory to prevent ANR
- **Repository:** OneKeyHQ/app-monorepo
- **PR Link:** <https://github.com/OneKeyHQ/app-monorepo/pull/10095>
- **Merged Date:** 2026-02-10
- **Category:** Native (+ Memory)

## Problem
A Sentry ANR on a Xiaomi Redmi 2201117TG (3.6GB RAM, Android 13) under critical memory pressure showed the main thread blocked in `ComponentActivity.onSaveInstanceState()`. That call serializes the View hierarchy, fragments and SavedStateRegistry. The work is wasted here because `super.onCreate(null)` already ignores saved state on restore, and React Native keeps its own state in JavaScript. No timing numbers are reported.

## Solution
- `MainActivity` tracks memory pressure with a volatile `isLowMemory` flag, updated in `onTrimMemory()` (true at `TRIM_MEMORY_RUNNING_LOW` or higher).
- `onSaveInstanceState()` returns early without calling `super` when the flag is set, so devices with enough memory keep the normal behavior.
- Safe only because the activity is created with `null` saved state; apps that restore from the bundle must not copy this.

## Code Pattern (Before vs. After)
```java
// Before
// (no override: ComponentActivity serializes views/fragments on every background)

// After
private volatile boolean isLowMemory = false;

@Override
public void onTrimMemory(int level) {
  super.onTrimMemory(level);
  isLowMemory = level >= ComponentCallbacks2.TRIM_MEMORY_RUNNING_LOW;
}

@Override
protected void onSaveInstanceState(@NonNull Bundle outState) {
  if (isLowMemory) {
    // Skip expensive state serialization to prevent ANR on low-memory devices.
    return;
  }
  super.onSaveInstanceState(outState);
}
```

## Takeaway
If the app never restores native saved state, skip serializing it when the system is under memory pressure to keep the main thread free during backgrounding.
