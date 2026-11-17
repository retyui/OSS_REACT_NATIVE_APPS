# perf: lazily render inactive tabs in City guide
- **Repository:** artsy/eigen
- **PR Link:** <https://github.com/artsy/eigen/pull/13972>
- **Merged Date:** 2026-08-24
- **Category:** Startup (+ Rendering)

## Problem
The Inbox and City guide screens use a tabs container that mounted every tab's screen up front on first render, even though only one tab is visible. That does unnecessary mounting and data work when the screen opens. The PR gives no measurements.

## Solution
- Pass the `lazy` prop to the tabs container in `Inbox` and `CityView`, so an inactive tab's screen only mounts when it is first visited.
- No other code changes; the PR changelog says it lazily mounts inactive tabs instead of mounting all up front.

## Code Pattern (Before vs. After)
```tsx
// Before
<TabsContainer
  initialTabName={initialPageName}
  onTabChange={({ tabName }) => handleNavigationTab(tabName)}
>

// After
<TabsContainer
  initialTabName={initialPageName}
  onTabChange={({ tabName }) => handleNavigationTab(tabName)}
  lazy
>
```

## Takeaway
Turn on lazy mounting for tab containers so hidden tabs do not cost render and fetch time on first open; check that tabs do not rely on being mounted early.
