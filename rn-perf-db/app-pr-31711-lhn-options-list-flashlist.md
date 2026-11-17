# [perf] Migrate LHNOptionList to FlashList
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/31711>
- **Merged Date:** 2023-11-22
- **Category:** Lists

## Problem
The left-hand navigation (LHN) chat list used `FlatList` with hand-tuned windowing (`initialNumToRender`, `maxToRenderPerBatch`, `windowSize`) and `getItemLayout`. The goal was smoother scrolling for accounts with many chats. This was a second attempt: the first FlashList migration (#31052) was reverted (#31642) after a regression that only showed on devices. The PR reports no numbers; testing was manual (scroll a long list and verify it is smooth).

## Solution
- Replace `FlatList` with `@shopify/flash-list`'s `FlashList` and drop `getItemLayout` and the window/batch props.
- Provide `estimatedItemSize` from the compact or regular row height constants, and `extraData={[currentReportID]}` so the highlighted row updates.
- Prefix the key extractor (`report_${item}`).
- `SidebarLinks` now flattens `contentContainerStyles` with `StyleSheet.flatten`. FlashList requires a plain object there.
- The loading skeleton is rendered as an absolutely positioned overlay only while the list is empty, so FlashList is always mounted with a measured parent.

## Code Pattern (Before vs. After)
```jsx
// Before
<FlatList
    keyExtractor={keyExtractor}
    renderItem={renderItem}
    getItemLayout={getItemLayout}
    initialNumToRender={20}
    maxToRenderPerBatch={5}
    windowSize={5}
    data={data}
    // ...
/>

// After
<FlashList
    keyExtractor={keyExtractor}
    renderItem={renderItem}
    estimatedItemSize={optionMode === CONST.OPTION_MODE.COMPACT ? variables.optionRowHeightCompact : variables.optionRowHeight}
    extraData={[currentReportID]}
    data={data}
    // ...
/>
```
```jsx
// SidebarLinks, After
<LHNOptionsList
    style={styles.flex1}
    contentContainerStyles={StyleSheet.flatten([styles.sidebarListContainer, {paddingBottom: StyleUtils.getSafeAreaMargins(insets).marginBottom}])}
    data={optionListItems}
    // ...
/>
```

## Takeaway
When moving a long, fixed-row-height list to FlashList, pass an accurate `estimatedItemSize`, use `extraData` for selection state, and flatten style arrays given to the list.
