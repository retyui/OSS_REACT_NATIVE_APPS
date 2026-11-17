# perf: memoize MessageRow
- **Repository:** RocketChat/Rocket.Chat.ReactNative
- **PR Link:** <https://github.com/RocketChat/Rocket.Chat.ReactNative/pull/7669>
- **Merged Date:** 2026-09-11
- **Category:** Lists (+ Rendering)

## Problem
`MessageRow` re-rendered once per visible cell on every incoming message, and every one of those renders was a no-op. The list's `renderItem` is a fresh closure each time, which invalidates `CellRenderer`, which calls the row again. Profiling an Android debug build showed about 19 to 21 row renders per sent message before, and 0 re-renders (1 mount) per send after, while `CellRenderer` still re-rendered 19 to 21 times.

## Solution
- Wrap `MessageRow` in `React.memo` with the default shallow comparator; its props are primitives and WatermelonDB model references, which compare correctly by reference.
- The date/unread separator was computed inside the row from the `lastSeen` context value, which would have defeated memoization. The calculation moves into a `useMessageSeparators` selector in the per-message zustand store, and `lastSeen` is passed down as a prop and mirrored into the store.
- Content updates still arrive via the message store tick, so message edits keep working.
- The inner function is named so the fiber shows as `Memo(MessageRow)` in the profiler. `memo` complements the React Compiler, which cannot stop a component being called.

## Code Pattern (Before vs. After)
```tsx
// Before
export const MessageRow = ({ item, previousItem, highlightedMessage, onLongPress }: TMessageRowProps) => {
	// ...
	const { lastSeen } = useRoomScreen();
	const { dateSeparator, showUnreadSeparator } = getMessageSeparators(item, previousItem, lastSeen);
	// ...
	return (
		<Message
			item={item}
			previousItem={previousItem}
			dateSeparator={dateSeparator}
			showUnreadSeparator={showUnreadSeparator}
			// ...
		/>
	);
};

// After
export const MessageRow = memo(function MessageRow({ item, previousItem, highlightedMessage, onLongPress }: TMessageRowProps) {
	// ...
	const { lastSeen } = useRoomScreen();
	// ...
	return (
		<Message
			item={item}
			previousItem={previousItem}
			lastSeen={lastSeen}
			withSeparators
			// ...
		/>
	);
});

// MessageStore.tsx
export const useMessageSeparators = (): TMessageSeparators =>
	useMessageStore(useShallow(s => getMessageSeparators(s.previousItem, s.item, s.lastSeen)));
```

## Takeaway
When a list item re-renders only because the parent recreates `renderItem`, wrap the row in `memo`, and first move any derived data that depends on changing context into a selector so props stay referentially stable.
