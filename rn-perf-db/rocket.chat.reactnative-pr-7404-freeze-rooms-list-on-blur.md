# fix: freeze when blurred to avoid off-screen re-renders on room open
- **Repository:** RocketChat/Rocket.Chat.ReactNative
- **PR Link:** <https://github.com/RocketChat/Rocket.Chat.ReactNative/pull/7404>
- **Merged Date:** 2026-06-23
- **Category:** Rendering

## Problem
On phones, opening a room pushes `RoomView` over `RoomsListView`, but the list underneath stays mounted and keeps rendering. Every off-screen `RoomItem` re-rendered during the transition, adding avoidable JS work to room open. The PR reports no numbers; the suggested check is a React profiler cold-open before/after.

## Solution
- Set `freezeOnBlur: true` on the `RoomsListView` screen in the phone Chats stack. react-native-screens/react-freeze then suspends rendering of that screen while it is covered.
- The component stays mounted, so the WatermelonDB subscription stays alive; only rendering is deferred and the list thaws with the latest data when the user returns.
- Tablet/master-detail is unaffected because the list is rendered as drawer content, a different path.

## Code Pattern (Before vs. After)
```tsx
// Before
<ChatsStack.Screen name='RoomsListView' component={RoomsListView} />

// After
<ChatsStack.Screen name='RoomsListView' component={RoomsListView} options={{ freezeOnBlur: true }} />
```

## Takeaway
`freezeOnBlur` on heavy list screens at the bottom of a stack is a one-line way to stop hidden screens from competing for the JS thread during navigation transitions.
