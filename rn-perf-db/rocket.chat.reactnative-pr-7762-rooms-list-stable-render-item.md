# fix: rooms list re-renders every row when a room opens
- **Repository:** RocketChat/Rocket.Chat.ReactNative
- **PR Link:** <https://github.com/RocketChat/Rocket.Chat.ReactNative/pull/7762>
- **Merged Date:** 2026-10-05
- **Category:** Lists (+ Rendering)

## Problem
`RoomsListView` subscribed to `state.room.subscribedRoom` and read it inside `onPressItem` and `renderItem`. Each room change therefore gave `renderItem` a new identity and every `RoomItem` a new `onPress`, so opening a room re-rendered every visible row of the list that was now hidden. The PR's React DevTools profile (opening a DM, iOS simulator, debug build) reports `RoomItem` renders going from 50 to 1, and the commit that re-rendered all rows (64 ms, 1,332 components) disappearing.

## Solution
- Read `subscribedRoom` from the Redux store at press time (`store.getState()`) instead of subscribing to it in render.
- Subscribe to `subscribedRoom` only in master-detail (tablet) mode, where it drives the selected-row highlight; on phones the selector returns `undefined`, so it never changes.
- Tablet highlight behavior is unchanged.

## Code Pattern (Before vs. After)
```tsx
// Before
const subscribedRoom = useAppSelector(state => state.room.subscribedRoom);
// ...
if (item.rid === subscribedRoom) {
	return;
}
// ...
isFocused={isMasterDetail && subscribedRoom === item.rid}

// After
const store = useStore<IApplicationState>();
const focusedRoom = useAppSelector(state => (isMasterDetail ? state.room.subscribedRoom : undefined));
// ...
if (item.rid === store.getState().room.subscribedRoom) {
	return;
}
// ...
isFocused={focusedRoom === item.rid}
```

## Takeaway
If a value is only needed inside an event handler, read it from the store at call time rather than subscribing, so the list's `renderItem` and row props stay stable.
