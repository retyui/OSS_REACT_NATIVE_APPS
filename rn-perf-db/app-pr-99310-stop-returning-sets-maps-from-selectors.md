# Perf: stop returning Sets and Maps from useOnyx collection selectors
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/99310>
- **Merged Date:** 2026-08-25
- **Category:** Rendering (+ Storage & IO)

## Problem
`useOnyx` compares selector results with `fast-equals`, which compares `Set`s and `Map`s by nested iteration (`areSetsEqual` / `areMapsEqual`) with no hash lookup. Every write to any member of a subscribed collection therefore cost O(n^2) comparisons. The PR gives the example that a Set of 5k report IDs is about 25M string compares per write, and a Map of report actions means O(n^2) full-object `deepEqual`s. No measured timings are reported.

## Solution
- Selectors that returned a `Set` now return `Record<string, true>` (policy report IDs) or `string[]` (transaction IDs), so equality is O(n) while lookups stay O(1).
- `MoneyRequestReportTransactionsNavigation` no longer builds a Map of every money-request action on three parent reports. The selector resolves only `{prevParentReportAction, nextParentReportAction}`, and the parent report IDs are deduped. The downstream `useMemo` became redundant and was deleted.
- `useTransactionViolation` takes `string[]` and builds its lookup `Set` internally, where `deepEqual` never sees it.

## Code Pattern (Before vs. After)
```tsx
// Before
const policyReportsSelector = useCallback(
    (reports: OnyxCollection<Report>) => {
        return Object.values(reports ?? {}).reduce((reportIDs, report) => {
            if (report?.policyID === policyID) {
                reportIDs.add(report.reportID);
            }
            return reportIDs;
        }, new Set<string>());
    },
    [policyID],
);
// ...
policyReports?.has(transaction.reportID)

// After
const policyReportsSelector = useCallback(
    (reports: OnyxCollection<Report>) => {
        const reportIDs: Record<string, true> = {};
        for (const report of Object.values(reports ?? {})) {
            if (report?.policyID === policyID) {
                reportIDs[report.reportID] = true;
            }
        }
        return reportIDs;
    },
    [policyID],
);
// ...
policyReports?.[transaction.reportID]
```

## Takeaway
Return plain objects or arrays from store selectors that are compared by deep equality; build `Set`/`Map` lookups inside the consumer, after the comparison.
