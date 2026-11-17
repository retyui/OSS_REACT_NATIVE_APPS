# perf(calendar): let React Compiler optimize the calendar screen
- **Repository:** mrzmyr/pixy-mood-tracker-app
- **PR Link:** <https://github.com/mrzmyr/pixy-mood-tracker-app/pull/807>
- **Merged Date:** 2026-10-18
- **Category:** Rendering

## Problem
The calendar screen opted out of React Compiler with `"use no memo"`. The opt-out guarded against a stale FlashList footer that only occurred while a screen was frozen (`freezeOnBlur`), but the tab bar that used it was gone and no navigator froze screens anymore. The opt-out therefore only cost renders: every screen render built a new footer element, so `memo(Calendar)` never hit. The PR gives no timings.

## Solution
- Remove the `"use no memo"` directive so the compiler memoizes the screen.
- Verify the premise: no `freezeOnBlur`, `enableFreeze` or `react-freeze` use, and only expo-router `Stack` navigators.
- Side effect fixed: `ForYouToday` relied on the uncompiled parent re-rendering at midnight. It now calls `useToday()` itself and passes `today` into `matchToday`, so the cached element still resets at midnight. A test covers this.
- `MISTAKES.md` is updated with the rule to run the e2e suite before freezing any screen.

## Code Pattern (Before vs. After)
```tsx
// Before
const CalendarScreenComponent = () => {
  /* Opt out of React Compiler. Compiled, this screen returns a cached footer ... */
  "use no memo";
  const colors = useColors();

// ForYouToday
const match = isEnabled ? matchToday(logState.items) : null; // reads clock at render

// After
const CalendarScreenComponent = () => {
  const colors = useColors();

// ForYouToday
// Own subscription: a memoized parent does not re-render at midnight.
const today = useToday();
const match = isEnabled ? matchToday(logState.items, today) : null;
```

## Takeaway
When enabling React Compiler on a screen, check children that read the clock or other non-React state at render time; give them their own subscription.
