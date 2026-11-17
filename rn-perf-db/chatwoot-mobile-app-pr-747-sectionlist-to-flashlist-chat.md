# feat: Replace SectionList with Flashlist in chat screen
- **Repository:** chatwoot/chatwoot-mobile-app
- **PR Link:** <https://github.com/chatwoot/chatwoot-mobile-app/pull/747>
- **Merged Date:** 2023-11-23
- **Category:** Lists (+ Rendering)

## Problem
The conversation screen rendered messages with a React Native `SectionList` that grouped messages by date and recomputed unique and grouped arrays inside the component on every render. The PR description is the empty template and reports no numbers; the intent evident from the diff is a faster, virtualized chat list.

## Solution
- Replaced `SectionList` with `@shopify/flash-list`'s `FlashList` (`inverted`, `estimatedItemSize={100}`, `onEndReachedThreshold={0.5}`).
- Dropped the per-render `findUniqueMessages` / `getGroupedMessages` work from the component. De-duplication and ordering now happen in the memoized Redux selector (`createDraftSafeSelector`), which sorts by `created_at`, reverses and dedupes by `id`.
- Wrapped the `Email` message component in `React.memo`.
- Side effect: the per-section date footers are no longer rendered in the list.

## Code Pattern (Before vs. After)
```tsx
// Before
const uniqueMessages = findUniqueMessages({allMessages});
const groupedMessages = getGroupedMessages({messages: uniqueMessages});
// ...
<SectionList
  inverted
  onEndReached={onEndReached}
  sections={groupedMessages}
  keyExtractor={(item, index) => item + index}
  renderItem={renderMessage}
  renderSectionFooter={({ section: { date } }) => <ChatMessageDate date={date} />}
/>

// After
<FlashList
  inverted
  data={allMessages}
  estimatedItemSize={100}
  renderItem={renderMessage}
  onEndReached={onEndReached}
  onEndReachedThreshold={0.5}
  ListFooterComponent={renderMoreLoader}
/>
// selector (memoized)
return completeMessages.reduce((acc, current) => {
  const x = acc.find(item => item.id === current.id);
  return x ? acc : acc.concat([current]);
}, []);
// Email.js
export default React.memo(EmailComponent);
```

## Takeaway
Use FlashList with an `estimatedItemSize` for long chat lists and move sorting and de-duplication into memoized selectors instead of render.
