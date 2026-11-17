# Migrate main chat to FlashList with inverted flag
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/85114>
- **Merged Date:** 2026-04-15
- **Category:** Lists

## Problem
The chat message list was rendered by a custom `InvertedFlatList` (FlatList plus a render-task queue and visible-content-position workarounds). Profiling the chat screen showed heavy render cost on mount and scroll for long conversations. In the PR's React profiler traces, the FlatList build had a total render duration of 2622 ms (p95 238.6 ms, p99 344 ms) across 74 commits.

## Solution
- Replaced `InvertedFlatList` with a new `InvertedFlashList` built on `@shopify/flash-list` 2.3.0 (view recycling instead of mounting one view per row).
- Added `getItemType={(item) => item.actionName}` so cells are recycled between rows of the same type, and `drawDistance={1500}`.
- Added `useFlashListScrollKey` to open a chat at a linked message, and a `CellRendererComponent` to keep z-index ordering correct.
- Removed the FlatList-only machinery (`RenderTaskQueue`, `shouldHideContent`, `shouldDisableVisibleContentPosition`, `initialNumToRender`) and hid the list during the initial scroll with a style instead.
- Six patches to FlashList were needed (inverted scroll on web, first item offset, measurement blocking with nested lists, inverted maintainVisibleContentPosition on Android).
- Caveat from the PR: Reassure shows higher render counts for FlashList because it renders in batches with measurement cycles. Real-world profiling is what the author used: the FlashList trace had a total render duration of 1819.7 ms (p95 121 ms, p99 166.5 ms) with the same commit count (74).

## Code Pattern (Before vs. After)
```tsx
// Before
<InvertedFlatList
    data={sortedVisibleReportActions}
    renderItem={renderItem}
    renderScrollComponent={renderActionSheetAwareScrollView}
    contentContainerStyle={[styles.chatContentScrollView, shouldFocusToTopOnMount ? styles.justifyContentEnd : undefined]}
    shouldHideContent={shouldScrollToEndAfterLayout}
    shouldDisableVisibleContentPosition={shouldScrollToEndAfterLayout}
    keyExtractor={keyExtractor}
    initialNumToRender={initialNumToRender}
    // ...
/>

// After
<InvertedFlashList
    data={sortedVisibleReportActions}
    renderItem={renderItem}
    keyExtractor={keyExtractor}
    drawDistance={1500}
    renderScrollComponent={renderActionSheetAwareScrollView}
    contentContainerStyle={[
        styles.chatContentScrollView,
        shouldFocusToTopOnMount && styles.justifyContentEnd,
        shouldScrollToEndAfterLayout && StyleUtils.getHiddenChatContentStyle(),
    ]}
    getItemType={(item) => item.actionName}
    initialScrollKey={linkedReportActionID}
    // ...
/>
```

## Takeaway
For long, heterogeneous feeds, move from FlatList to FlashList and give it a `getItemType` so recycling works per row type. Expect to maintain patches if the list is inverted or needs scroll-position anchoring.
