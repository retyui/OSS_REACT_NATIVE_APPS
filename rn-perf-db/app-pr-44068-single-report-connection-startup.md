# perf: improve native apps startup
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/44068>
- **Merged Date:** 2024-07-08
- **Category:** Startup (+ Storage & IO)

## Problem
Many util files each opened their own `Onyx.connect` on the `report_` collection (14 connections). With around 15k reports, every connection does heavy work per update (e.g. recursive `fastMerge`), which does not scale; the PR says this took around 18 seconds in total on large data. Separately, `useReportID` called the heavy `SidebarUtils.getOrderedReportID` three times during startup, costing around 3 seconds per the PR.

## Solution
- Add one shared `ReportConnection` module that holds the single reports subscription and exports `getAllReports()`. Other utils call it instead of keeping their own `allReports` copy; callbacks that need to react are exported and invoked from the shared connection.
- Memoize `policyMemberAccountIDs` and expose it through the `ReportIDs` context, removing one redundant ordering call.
- Skip recomputing ordered report IDs when `currentReportID` is the default `-1`.
- Smaller fixes: avoid `fastMerge` in `getLastVisibleAction` when there are no `actionsToMerge`, and replace sort-then-pick with `Array.find` in `isActionableJoinRequestPending`.

## Code Pattern (Before vs. After)
```ts
// Before (repeated in each util file)
let allReports: OnyxCollection<Report>;
Onyx.connect({
    key: ONYXKEYS.COLLECTION.REPORT,
    waitForCollectionCallback: true,
    callback: (value) => (allReports = value),
});

function getCustomUnitRateID(reportID: string) {
    const report = allReports?.[`${ONYXKEYS.COLLECTION.REPORT}${reportID}`];
    // ...
}

// After
import * as ReportConnection from './ReportConnection';

function getCustomUnitRateID(reportID: string) {
    const allReports = ReportConnection.getAllReports();
    const report = allReports?.[`${ONYXKEYS.COLLECTION.REPORT}${reportID}`];
    // ...
}
```

```tsx
// Before
if (derivedCurrentReportID && !orderedReportIDs.includes(derivedCurrentReportID)) {
    return {orderedReportIDs: getOrderedReportIDs(derivedCurrentReportID), currentReportID: derivedCurrentReportID ?? '-1'};
}

// After
if (derivedCurrentReportID && derivedCurrentReportID !== '-1' && orderedReportIDs.indexOf(derivedCurrentReportID) === -1) {
    return {orderedReportIDs: getOrderedReportIDs(derivedCurrentReportID), currentReportID: derivedCurrentReportID ?? '-1', policyMemberAccountIDs};
}
```

## Takeaway
Keep one subscription per large store collection and share it through a getter; N subscriptions multiply the per-update work by N.
