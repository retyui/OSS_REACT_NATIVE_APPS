# Improve emoji picker performance introducing flashList
- **Repository:** mattermost/mattermost-mobile
- **PR Link:** <https://github.com/mattermost/mattermost-mobile/pull/8650>
- **Merged Date:** 2025-03-11
- **Category:** Lists (+ Images)

## Problem
The emoji picker was built on `SectionList` with a precomputed `getItemLayout`. Its slow first render meant a screenshot-protection feature on iOS sometimes did not find the view to protect, and on Android, searching and loading custom emojis eventually crashed the app. The PR reports no benchmarks, only that after the change the first render was fast enough for the view to always be found.

## Solution
- Replaced `SectionList`/`BottomSheetSectionList` with `FlashList`/`BottomSheetFlashList` for both the section list and the search results list. Sections and emoji rows are flattened into a single typed data array, with `getItemType` separating headers from rows so cells recycle per type, and `estimatedItemSize` set from the emoji size.
- Sticky category headers use `stickyHeaderIndices`; a patch to FlashList (`@shopify/flash-list` 1.7.3) adds an `onStickyHeaderIndexChanged` callback to keep the category bar in sync while scrolling.
- Custom emoji images are prefetched when their data is fetched (`expo-image` prefetch to disk on iOS, `Image.prefetch` on Android).
- On Android, emoji images moved back to the React Native `Image` (Fresco) because the Glide-based libraries tried still crashed; this is a crash fix rather than a speed gain.

## Code Pattern (Before vs. After)
```tsx
// Before
const List = useMemo(() => (isTablet ? SectionList : BottomSheetSectionList), [isTablet]);
// ...
<List
    getItemLayout={getItemLayout}
    onEndReached={onLoadMoreCustomEmojis}
    onEndReachedThreshold={2}
    renderItem={renderItem}
    renderSectionHeader={renderSectionHeader}
    sections={sections}
    stickySectionHeadersEnabled={true}
/>

// After
const getItemType = (item: SectionListItem) => item.type;
const List = useMemo(() => (isTablet ? FlashList : BottomSheetFlashList), [isTablet]);
// ...
<List
    data={sections}
    estimatedItemSize={EMOJI_SIZE + EMOJI_ROW_MARGIN}
    getItemType={getItemType}
    keyExtractor={keyExtractor}
    onEndReachedThreshold={0.5}
    onEndReached={loadMoreCustomEmojis}
    onStickyHeaderIndexChanged={handleStickyHeaderIndexChanged}
    renderItem={renderItem}
    stickyHeaderIndices={stickyHeaderIndices}
/>
```

```ts
// After: prefetch custom emoji images
export function prefetchCustomEmojiImages(client: Client, emojis: CustomEmoji[]) {
    if (Platform.OS === 'ios') {
        ExpoImage.prefetch(emojis.map((ce) => client.getCustomEmojiImageUrl(ce.id)), 'disk');
    } else {
        emojis.forEach((ce) => {
            Image.prefetch(client.getCustomEmojiImageUrl(ce.id));
        });
    }
}
```

## Takeaway
For long grouped grids, flatten sections into one typed array and use FlashList with `getItemType`, instead of SectionList plus hand-computed layouts.
