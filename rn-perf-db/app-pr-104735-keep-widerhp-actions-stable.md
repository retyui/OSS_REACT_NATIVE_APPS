# perf: Keep WideRHP actions stable on row press
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/104735>
- **Merged Date:** 2026-10-23
- **Category:** Rendering

## Problem
Pressing a report row in Search (or a report preview, transaction-thread link, etc.) calls `markReportRHPWidth` before navigating. The width hints lived in `useState` inside `WideRHPContextProvider`, so every mark created new action functions and every `useWideRHPActions()` consumer re-rendered on each row press. Every list row is one of those consumers. The provider also subscribed to the whole `report_` collection just to skip the wide hint for invoices and tasks. No timings are reported.

## Solution
- Moved the hints into a `useRef` Map. They are only read in `useRHPWidth` effects and callers mark before navigating, so the target screen reads the hint on mount. The actions object stays stable across presses.
- Dropped the provider's `expenseReportSelector` subscription to the whole report collection.
- Moved the invoice/task exclusion into `useRHPWidth`, which now reads only the open report's `type` via a new `getReportType` selector. Caveat from the PR: if the report has not loaded yet, the RHP now corrects to narrow once its type arrives, where before it stayed wide.

## Code Pattern (Before vs. After)
```tsx
// Before
const [reportRHPWidthHints, setReportRHPWidthHints] = useState<Map<string, RHPWidthHint>>(() => new Map());
const [allReports] = useOnyx(ONYXKEYS.COLLECTION.REPORT, {selector: expenseReportSelector});
// ...
const markReportRHPWidth = (reportID, width) => {
    // ...
    setReportRHPWidthHints((prev) => {
        if (prev.get(reportID) === width) {
            return prev;
        }
        const next = new Map(prev);
        next.set(reportID, width);
        return next;
    });
};

// After
const reportRHPWidthHintsRef = useRef(new Map<string, RHPWidthHint>());
// ...
const markReportRHPWidth = (reportID, width) => {
    if (!reportID) {
        return;
    }
    reportRHPWidthHintsRef.current.set(reportID, width);
};

// useRHPWidth
const [reportType] = useOnyx(`${ONYXKEYS.COLLECTION.REPORT}${getNonEmptyStringOnyxID(reportID)}`, {selector: getReportType});
const isWideHintExcluded = reportType === CONST.REPORT.TYPE.INVOICE || reportType === CONST.REPORT.TYPE.TASK;
```

## Takeaway
Data that is only read in effects or callbacks belongs in a ref, not state, especially in a context whose actions are consumed by every list row.
