# perf(mobile): skip inbox row re-renders on unchanged polls
- **Repository:** elie222/rakazo
- **PR Link:** <https://github.com/elie222/rakazo/pull/1296>
- **Merged Date:** 2026-10-10
- **Category:** Rendering (+ Lists)

## Problem
The inbox polls `spaces/list` and `me` every 5 s while focused. Each response produced fresh arrays and objects, `renderItem` was an inline function, and `BotRow`/`GroupRow`/`SearchRow` were not memoized, so every mounted row re-rendered every 5 s even when nothing changed. In a React DOM harness with 30 mounted bots and 12 identical polls (about 1 idle minute), row renders went from 360 to 0; when one bot's status changed, from 30 to 1.

## Solution
- `retainInboxValue(current, next)` deep-compares a response against current state and returns the previous references for structurally equal branches, matching list items by `id`. It is applied in the `setBots`/`setGroups`/`setSpaces`/`setMe` updaters.
- Row components are wrapped in `memo` with stable callback props (they receive the item, not per-row closures) and `renderItem` is a `useCallback`.
- A `day` value is passed to rows and refreshed at midnight and on foreground, so relative timestamps stay correct despite memoization.
- Polling cadence and visuals are unchanged.

## Code Pattern (Before vs. After)
```tsx
// Before
setBots(navigation.current.bots);
setGroups(navigation.current.groups);
setMe(nextMe);
// ...
function BotRow({ bot, depth = 0, onPress, onLongPress }: {...}) { /* ... */ }
// ...
renderItem={({ item }) => item.type === "search" ? (<SearchRow hit={item.hit} onPress={() => {/* inline */}} />) : /* ... */}

// After
setBots((current) => retainInboxValue(current, navigation.current.bots));
setGroups((current) => retainInboxValue(current, navigation.current.groups));
setMe((current) => retainInboxValue(current, nextMe));
// ...
const BotRow = memo(function BotRow({ bot, day, depth = 0, onPress, onLongPress }: {
  onPress: (bot: MobileBot | SpaceBot) => void;
  onLongPress?: (bot: MobileBot | SpaceBot) => void;
  /* ... */
}) { /* ... */ });
// ...
const renderItem = useCallback(
  ({ item }: { item: InboxItem }) =>
    item.type === "search" ? <SearchRow hit={item.hit} onPress={openSearchHit} /> : /* ... */,
  [/* stable deps */],
);

// inbox-retention.ts (core)
if (current === next) return current;
if (Array.isArray(current) && Array.isArray(next)) {
  const byId = new Map(current.filter(hasId).map((item) => [item.id, item]));
  const retained = next.map((item, index) =>
    retainInboxValue(hasId(item) ? byId.get(item.id) : current[index], item),
  );
  return (current.length === retained.length &&
    retained.every((item, index) => item === current[index]) ? current : retained) as T;
}
```

## Takeaway
When polling returns equal data, keep the old object references (structural sharing) and memoize rows with stable props so unchanged rows skip rendering.
