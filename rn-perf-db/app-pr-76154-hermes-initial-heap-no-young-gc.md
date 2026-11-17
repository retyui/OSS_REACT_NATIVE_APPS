# [NO QA] perf: Increase initial Hermes heap size for faster startup and delay Young-Gen GC
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/76154>
- **Merged Date:** 2026-01-26
- **Category:** Startup (+ Memory)

## Problem
Hermes starts with a 4 MB initial heap; once it fills, garbage collection runs and slows down startup work. The PR reports that raising the initial heap and delaying Young-Gen GC saves roughly 150-300 ms of startup on low-end Android devices (it varies with device RAM and memory pressure).

## Solution
Two `patch-package` patches against `HermesInstance.cpp` in React Native 0.81.4:
- set `withInitHeapSize` to 150 MB and disable releasing unused memory;
- force the existing `allocInOldGenBeforeTTI` branch on, so early allocations go straight to the old generation and Young-Gen GC resumes after the first Old-Gen GC.
The stated downside is higher RAM use at startup. The patches are not intended for upstream.

## Code Pattern (Before vs. After)
```cpp
// Before
auto gcConfig = ::hermes::vm::GCConfig::Builder()
                    // Default to 3GB
                    .withMaxHeapSize(3072 << 20)
                    .withName("RNBridgeless");

if (allocInOldGenBeforeTTI) {

// After
auto gcConfig = ::hermes::vm::GCConfig::Builder()
                    .withInitHeapSize(150ll * 1024 * 1024)
                    .withShouldReleaseUnused(::hermes::vm::kReleaseUnusedNone)
                    // Default to 3GB
                    .withMaxHeapSize(3072 << 20)
                    .withName("RNBridgeless");

// Old-Gen GC enables Young-Gen GC again
if (/*allocInOldGenBeforeTTI*/ true) {
```
