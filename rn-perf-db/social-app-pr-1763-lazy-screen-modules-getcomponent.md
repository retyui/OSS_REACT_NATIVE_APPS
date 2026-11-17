# Initialize screen modules lazily
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/1763>
- **Merged Date:** 2023-10-27
- **Category:** Startup

## Problem
Navigation.tsx referenced every screen component in JSX, so with inline requires all screen modules were still initialized at startup even though only one screen renders at a time. Any screen that does work at module-init time would add to startup cost. The PR shows profiler screenshots (before: all screen modules block startup; after: only the initial screen initializes) but gives no numbers in text.

## Solution
- Replace `component={Screen}` with React Navigation's `getComponent={() => Screen}` on every `Stack.Screen`.
- The screen identifier is only evaluated when the screen is first rendered, so with Metro inline requires its module initializes lazily as the user navigates.
- No async gap is introduced; the author first tried `React.lazy` and then switched to the built-in API.

## Code Pattern (Before vs. After)
```tsx
// Before
<Stack.Screen
  name="Settings"
  component={SettingsScreen}
  options={{title: title('Settings')}}
/>

// After
<Stack.Screen
  name="Settings"
  getComponent={() => SettingsScreen}
  options={{title: title('Settings')}}
/>
```

## Takeaway
With inline requires enabled, use `getComponent` for navigator screens so unvisited screens are not loaded at startup.
