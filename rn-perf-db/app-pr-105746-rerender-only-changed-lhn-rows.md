# perf: Re-render only the LHN rows a report switch changes
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/105746>
- **Merged Date:** 2026-11-05
- **Category:** Rendering (+ Lists)

## Problem
Every LHN row read the current report ID from a context that also held the report open in the RHP. Opening a report re-rendered every row, and so did opening or closing anything in the RHP, although only two rows change on a report switch and none for the RHP. Measured on web (dev build, 20 rows): rows re-rendered per LHN tap 20 to 22 down to 2 to 4, and per RHP open or close 20 down to 0. Row render time per tap went from 3.8 ms to 3.0 ms, and Reassure `SidebarLinksCurrentReportChange` (500 reports) from 51.7 ms to 25.6 ms. The author notes the live-app time saved is small and `ManualOpenReport` does not move.

## Solution
- Split the RHP report ID into its own `CurrentRHPReportIDContext`, so RHP changes no longer reach the rows.
- Added a small external store plus `useIsCurrentReportID(reportID)` built on `useSyncExternalStore`; each row subscribes to a boolean for its own report, so only the row that lost focus and the row that gained it re-render.
- The current report ID is still set inside `startTransition`; the store is synced in a `useLayoutEffect` so the two rows update in the same frame.
- Caveat in the code: store updates cannot be transitions, so subscribers re-render synchronously and must stay cheap; do not mix `useIsCurrentReportID` and `useCurrentReportIDState` in one component, since they disagree for one commit.

## Code Pattern (Before vs. After)
```tsx
// Before
const {currentReportID: currentReportIDValue} = useCurrentReportIDState();
// ...
const isReportFocused = isOptionFocused && currentReportIDValue === reportID;

// After
const isCurrentReport = useIsCurrentReportID(reportID);
// ...
const isReportFocused = isOptionFocused && isCurrentReport;

// useCurrentReportID.tsx
useLayoutEffect(() => {
    store.set(currentReportID);
}, [store, currentReportID]);

function useIsCurrentReportID(reportID: string) {
    const store = useContext(CurrentReportIDStoreContext);
    return useSyncExternalStore(store.subscribe, () => store.get() === reportID);
}
```

## Takeaway
When many list rows need to know "am I the selected item", subscribe each to a derived boolean through an external store instead of reading the selected ID from context.
