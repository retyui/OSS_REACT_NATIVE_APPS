# perf: Memoize LHN row option data by moving getOptionData into a child component
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/104860>
- **Merged Date:** 2026-10-28
- **Category:** Rendering (+ Lists)

## Problem
`OptionRowLHNData` called `SidebarUtils.getOptionData(...)` after all its hooks, and React Compiler did not memoize that call. Each render created a new `optionItem` and re-rendered the whole `OptionRowLHN` subtree, even when nothing on the row changed. Rows re-render often because each reads `currentReportID` from context. Measured on web (profiling build, 2x CPU throttling, 20 visible rows), total LHN row render time per interaction: open a report 17.0 ms to 4.2 ms, open report details in the RHP 15.1 ms to 2.5 ms, close the RHP 15.0 ms to 2.9 ms. A new Reassure test (`SidebarLinksCurrentReportChange`, 500 reports) went from 103.0 ms to 44.9 ms. The `ManualOpenReport` and `ManualNavigateToInboxTab` spans did not move.

## Solution
- Moved `getOptionData`, the single-sender IOU avatar trim, the placeholder row and `shouldShowMarkAsDone` into a new `OptionRowLHNContent` child. `OptionRowLHNData` keeps the hooks and passes one `optionDataParams` object. In the child the compiler memoizes the params, the element and the `getOptionData` call.
- Because the call is now memoized, `getOptionData` can no longer read Onyx data it was not given. Pending join requests and `pendingChatMembers`/`pendingExpenseAction` are now passed in (`hasPendingJoinRequest`, and `reportMetadata` from a new per-report subscription with a two-field selector). Callers that do not pass them fall back to the old reads.

## Code Pattern (Before vs. After)
```tsx
// Before (OptionRowLHNData)
// React Compiler auto-memoizes each expression ... (comment claimed this was memoized)
const optionItem = SidebarUtils.getOptionData({
    dateFnsLocale,
    report: fullReport,
    // ... ~30 params
});
// ...
return <OptionRowLHN {...propsToForward} optionItem={finalOptionItem} /* ... */ />;

// After (OptionRowLHNData)
const hasPendingJoinRequest = Object.values(reportActions ?? {}).some(isActionableJoinRequestPendingReportAction);
return (
    <OptionRowLHNContent
        {...propsToForward}
        viewMode={viewMode}
        isReportFocused={isReportFocused}
        optionDataParams={{
            dateFnsLocale,
            report: fullReport,
            // ...
            reportMetadata,
            hasPendingJoinRequest,
            lastActionTransaction,
        }}
    />
);

// OptionRowLHNContent: kept apart because the compiler does not memoize getOptionData in the parent
function OptionRowLHNContent({optionDataParams, isReportFocused, viewMode, ...rest}) {
    const optionItem = SidebarUtils.getOptionData(optionDataParams);
    // ...
}
```

## Takeaway
If a React Compiler component does not memoize an expensive call, check the output; moving the call into a child that receives a single params object can let the compiler cache it. Pass in any data the memoized function used to read implicitly.
